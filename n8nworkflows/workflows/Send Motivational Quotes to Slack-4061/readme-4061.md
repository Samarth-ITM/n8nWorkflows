Send Motivational Quotes to Slack

https://n8nworkflows.xyz/workflows/send-motivational-quotes-to-slack-4061


# Send Motivational Quotes to Slack

### 1. Workflow Overview

This workflow is designed to automatically send **3 motivational quotes per day** to a specified public Slack channel. It leverages the free ZenQuotes API to fetch random inspirational quotes, formats them neatly, and posts them in Slack to boost team morale and productivity.  

The workflow is logically divided into these blocks:

- **1.1 Scheduled Triggers:** Three separate schedule trigger nodes initiate the workflow at 08:00, 13:00, and 18:00 daily.
- **1.2 Quote Retrieval:** A single HTTP Request node fetches a random motivational quote from the ZenQuotes API.
- **1.3 Message Formatting:** A Code node formats the quote and author into a clean text string.
- **1.4 Slack Posting:** A Slack node posts the formatted quote message to a chosen Slack channel.

This modular design allows easy adjustment of trigger times, message format, and Slack channel, making it ideal for daily team inspiration.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Triggers

- **Overview:**  
  This block contains three independent schedule trigger nodes, each responsible for starting the workflow once daily at a specific time: morning (8:00), midday (13:00), and evening (18:00). They ensure the quotes are sent consistently throughout the workday.

- **Nodes Involved:**  
  - 08:00 – Morning Boost  
  - 13:00 – Midday Reminder  
  - 18:00 – Evening Motivation

- **Node Details:**

  - **08:00 – Morning Boost**
    - Type: Schedule Trigger  
    - Role: Triggers workflow daily at 08:00 (with minute set to 8 for slight offset)  
    - Configuration: Interval trigger at hour 8, minute 8  
    - Inputs: None (trigger node)  
    - Outputs: Connects to "Fetch Daily Quote (ZenQuotes API)"  
    - Edge Cases: Workflow will not trigger if n8n is offline or the node is deactivated.  
    - Version: 1.2  

  - **13:00 – Midday Reminder**
    - Type: Schedule Trigger  
    - Role: Triggers workflow daily at 13:00  
    - Configuration: Interval trigger at hour 13, minute default 0  
    - Inputs: None  
    - Outputs: Connects to "Fetch Daily Quote (ZenQuotes API)"  
    - Edge Cases: Same as above.  
    - Version: 1.2  

  - **18:00 – Evening Motivation**
    - Type: Schedule Trigger  
    - Role: Triggers workflow daily at 18:00  
    - Configuration: Interval trigger at hour 18, minute default 0  
    - Inputs: None  
    - Outputs: Connects to "Fetch Daily Quote (ZenQuotes API)"  
    - Edge Cases: Same as above.  
    - Version: 1.2  

---

#### 2.2 Quote Retrieval

- **Overview:**  
  This block fetches a random motivational quote from the ZenQuotes API. It's called by each schedule trigger and is the source of daily inspirational content.

- **Nodes Involved:**  
  - Fetch Daily Quote (ZenQuotes API)

- **Node Details:**

  - **Fetch Daily Quote (ZenQuotes API)**
    - Type: HTTP Request  
    - Role: Fetches a random quote JSON from https://zenquotes.io/api/random  
    - Configuration:  
      - HTTP Method: GET (default)  
      - URL: https://zenquotes.io/api/random  
      - No authentication or API key required  
      - No additional headers or body parameters  
    - Key Expressions: None; raw JSON response expected  
    - Inputs: Connected from all three schedule triggers  
    - Outputs: Passes retrieved quote JSON to "Format Slack Message"  
    - Edge Cases:  
      - Network timeouts or API downtime may cause failure  
      - Unexpected API response format (e.g., if API changes response structure)  
      - Rate limiting unlikely but possible if called excessively  
    - Version: 4.2  

---

#### 2.3 Message Formatting

- **Overview:**  
  This block formats the quote and author into a single string suitable for Slack message posting.

- **Nodes Involved:**  
  - Format Slack Message

- **Node Details:**

  - **Format Slack Message**
    - Type: Code  
    - Role: Converts raw API quote data into a text string formatted as: `{quote} — {author}`  
    - Configuration:  
      - JavaScript code snippet:  
        ```js
        return [
          {
            json: {
              text: `${$json["q"]} — ${$json["a"]}`
            }
          }
        ];
        ```  
      - Assumes API response JSON fields `q` = quote, `a` = author  
    - Inputs: From "Fetch Daily Quote (ZenQuotes API)"  
    - Outputs: To "Send to Slack Channel"  
    - Edge Cases:  
      - If API response lacks expected fields `q` or `a`, the message string may be malformed or empty  
      - Expression errors if input JSON structure changes  
    - Version: 2  

---

#### 2.4 Slack Posting

- **Overview:**  
  This block posts the formatted motivational quote message to a specified Slack channel.

- **Nodes Involved:**  
  - Send to Slack Channel

- **Node Details:**

  - **Send to Slack Channel**
    - Type: Slack  
    - Role: Posts the text message to a public Slack channel  
    - Configuration:  
      - Text: uses expression `={{$json["text"]}}` to get message from previous node  
      - Channel selection: Uses a channelId parameter in list mode (user selects channel in UI)  
      - Other options: disables "include link to workflow"  
    - Inputs: From "Format Slack Message"  
    - Outputs: None (end node)  
    - Credential Requirements: Slack credentials with scopes:  
      - `chat:write`  
      - `chat:write.public`  
      - `channels:read`  
    - Edge Cases:  
      - Authentication failure if Slack token invalid or expired  
      - Slack API rate limits or channel permissions errors  
      - Channel ID must be valid and accessible by the bot  
    - Version: 2.3  

---

### 3. Summary Table

| Node Name                     | Node Type          | Functional Role                  | Input Node(s)                            | Output Node(s)              | Sticky Note                                                                 |
|-------------------------------|--------------------|---------------------------------|-----------------------------------------|-----------------------------|-----------------------------------------------------------------------------|
| 08:00 – Morning Boost          | Schedule Trigger   | Morning daily trigger            | None                                    | Fetch Daily Quote (ZenQuotes API) |                                                                             |
| 13:00 – Midday Reminder        | Schedule Trigger   | Midday daily trigger             | None                                    | Fetch Daily Quote (ZenQuotes API) |                                                                             |
| 18:00 – Evening Motivation     | Schedule Trigger   | Evening daily trigger            | None                                    | Fetch Daily Quote (ZenQuotes API) |                                                                             |
| Fetch Daily Quote (ZenQuotes API) | HTTP Request       | Fetch random motivational quote | 08:00 – Morning Boost, 13:00 – Midday Reminder, 18:00 – Evening Motivation | Format Slack Message         |                                                                             |
| Format Slack Message           | Code               | Format quote text for Slack      | Fetch Daily Quote (ZenQuotes API)       | Send to Slack Channel        |                                                                             |
| Send to Slack Channel          | Slack              | Post message to Slack channel    | Format Slack Message                     | None                        | ⚙️ Setup Steps: Create Slack App with `chat:write`, `chat:write.public`, `channels:read` scopes; use bot token or OAuth credentials |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Schedule Trigger Nodes**  
   - Add three Schedule Trigger nodes.  
   - Name them:  
     - "08:00 – Morning Boost"  
     - "13:00 – Midday Reminder"  
     - "18:00 – Evening Motivation"  
   - Configure triggers:  
     - Morning Boost: Interval trigger, daily at 08:08 (hour 8, minute 8)  
     - Midday Reminder: Interval trigger, daily at 13:00 (hour 13, minute 0)  
     - Evening Motivation: Interval trigger, daily at 18:00 (hour 18, minute 0)  

2. **Create the HTTP Request Node**  
   - Add an HTTP Request node named "Fetch Daily Quote (ZenQuotes API)".  
   - Set Method to GET.  
   - Set URL to `https://zenquotes.io/api/random`.  
   - No authentication or additional headers needed.  
   - Connect outputs of all three Schedule Trigger nodes to this node's input.

3. **Create the Code Node for Formatting**  
   - Add a Code node named "Format Slack Message".  
   - Use the following JavaScript code snippet:  
     ```js
     return [
       {
         json: {
           text: `${$json["q"]} — ${$json["a"]}`
         }
       }
     ];
     ```  
   - Connect the output of "Fetch Daily Quote (ZenQuotes API)" to input of this Code node.

4. **Create the Slack Node**  
   - Add a Slack node named "Send to Slack Channel".  
   - Configure the text parameter with expression: `={{$json["text"]}}`.  
   - Set "Select" option to "channel".  
   - Select your target public Slack channel or use the channelId parameter in list mode to pick from a dropdown.  
   - Disable "Include link to workflow" for a clean message.  
   - Connect the output of "Format Slack Message" to this Slack node.

5. **Credentials Setup**  
   - Create Slack credentials in n8n:  
     - Use Bot Token (`xoxb-...`) from your Slack App with scopes:  
       - `chat:write`  
       - `chat:write.public`  
       - `channels:read`  
     - Alternatively, set up OAuth2 credentials if you have HTTPS and a domain (optional).  
   - Assign these credentials to the Slack node.

6. **Activate the Workflow**  
   - Review connections and node parameters.  
   - Activate the workflow in n8n.  
   - The workflow will now run automatically at the three scheduled times, fetching and posting motivational quotes.

---

### 5. General Notes & Resources

| Note Content                                                                                     | Context or Link                                                  |
|-------------------------------------------------------------------------------------------------|-----------------------------------------------------------------|
| This workflow uses the free ZenQuotes API which requires no API key.                            | https://zenquotes.io/                                           |
| Slack app scopes required: `chat:write`, `chat:write.public`, `channels:read`                   | https://api.slack.com/apps                                      |
| Slack credentials should be bot tokens (`xoxb-...`), easiest to set up for this bot             | Slack API documentation                                         |
| Workflow designed by TuguiDragos.com, includes a helpful setup guide and screenshot            | https://tuguidragos.com/n8n-slack-daily-motivation-bot         |
| The workflow is ideal for boosting team morale with automated daily inspiration                  | Use cases: productivity, team motivation, daily check-ins       |

---

**Disclaimer:** The content provided is derived exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and does not contain any illegal, offensive, or protected elements. All data handled are legal and public.