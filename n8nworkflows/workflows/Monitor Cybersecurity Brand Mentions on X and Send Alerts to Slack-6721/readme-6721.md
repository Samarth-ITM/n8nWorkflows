Monitor Cybersecurity Brand Mentions on X and Send Alerts to Slack

https://n8nworkflows.xyz/workflows/monitor-cybersecurity-brand-mentions-on-x-and-send-alerts-to-slack-6721


# Monitor Cybersecurity Brand Mentions on X and Send Alerts to Slack

### 1. Workflow Overview

This workflow is designed to monitor X (formerly Twitter) for cybersecurity-related brand mentions, vulnerability identifiers, and threat keywords, and to send real-time alerts to a designated Slack channel. It targets cybersecurity teams, IT/security operations, brand managers, and SMEs who need timely awareness of relevant online discussions impacting security or reputation.

The workflow is logically divided into these blocks:

- **1.1 Scheduling & Input Reception:** A schedule trigger initiates periodic polling of X for tweets matching specified cybersecurity-related keywords.
- **1.2 Data Capture & Formatting:** Tweets fetched are processed to extract key information and formatted into a clear notification message.
- **1.3 Filtering:** Basic filtering is applied to exclude irrelevant or spammy mentions (e.g., tweets containing "bot").
- **1.4 Notification Delivery:** Valid mentions trigger sending formatted alerts directly to a configured Slack channel.
- **1.5 Workflow Termination:** An end node marks clean completion of the workflow execution path.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduling & Input Reception

- **Overview:** This block triggers the workflow on a recurring schedule and fetches recent tweets matching defined cybersecurity keywords.
- **Nodes Involved:** `Schedule Trigger`, `Monitor: Cybersecurity Keywords`

##### Node: Schedule Trigger
- **Type & Role:** Schedule Trigger node; initiates workflow execution periodically.
- **Configuration:** Uses default interval settings (runs every minute by default, as implied by empty interval array).
- **Expressions/Variables:** None.
- **Input/Output:** No input; output to `Monitor: Cybersecurity Keywords`.
- **Version Requirements:** Version 1.2.
- **Potential Failures:** Scheduling misconfiguration; node downtime.
- **Sub-workflow:** None.

##### Node: Monitor: Cybersecurity Keywords (Twitter)
- **Type & Role:** Twitter node; performs search operation on X.
- **Configuration:** Searches for tweets matching the query string:
  ```
  ="[YourBrandName]" OR "CVE-2024-XXXX" OR "zeroday"
  ```
  Limits results to 10 tweets per execution.
- **Expressions/Variables:** The search query is hardcoded but intended to be customized with actual brand names or CVE identifiers.
- **Input/Output:** Input from `Schedule Trigger`; output to `Format Notification`.
- **Version Requirements:** Version 1.
- **Potential Failures:** Twitter API rate limits, authentication errors, query syntax errors, network timeouts.
- **Sub-workflow:** None.

---

#### 2.2 Data Capture & Formatting

- **Overview:** Extracts relevant tweet details and constructs a human-readable notification message.
- **Nodes Involved:** `Format Notification`

##### Node: Format Notification (Code)
- **Type & Role:** Code node; processes raw tweet JSON to create notification text.
- **Configuration:** Custom JavaScript code extracts tweet text, username, and constructs a tweet URL. Optionally, can filter out unwanted tweets.
- **Key Expressions:**
  ```js
  const tweet = $json.text;
  const user = $json.user.screen_name;
  const tweetUrl = `https://twitter.com/${user}/status/${$json.id_str}`;
  return [{
    json: {
      ...$json,
      notificationMessage: `🚨 New Cybersecurity Mention! 🚨\nUser: @${user}\nTweet: ${tweet}\nLink: ${tweetUrl}`
    }
  }];
  ```
- **Input/Output:** Receives tweet data from `Monitor: Cybersecurity Keywords`; outputs data with added `notificationMessage` to `Valid Mention?`.
- **Version Requirements:** Version 1.
- **Potential Failures:** Expression errors if input JSON structure changes; runtime errors in JavaScript code.
- **Sub-workflow:** None.

---

#### 2.3 Filtering

- **Overview:** Applies a simple filter to exclude tweets containing unwanted keywords to reduce noise.
- **Nodes Involved:** `Valid Mention?`

##### Node: Valid Mention? (If)
- **Type & Role:** Conditional node; checks if the notification message contains the word `"bot"`.
- **Configuration:** Evaluates condition:
  - `notificationMessage` does NOT contain `"bot"` (case-sensitive).
- **Expressions/Variables:** Uses expression:
  ```
  {{$json.notificationMessage}}
  ```
  to check for substring.
- **Input/Output:** Input from `Format Notification`; if true, routes to `Send Notification`; else routes to `End Workflow`.
- **Version Requirements:** Version 2.2.
- **Potential Failures:** Expression evaluation errors if `notificationMessage` is missing or malformed.
- **Sub-workflow:** None.

---

#### 2.4 Notification Delivery

- **Overview:** Sends the formatted notification message to a Slack channel for alerting the team.
- **Nodes Involved:** `Send Notification`

##### Node: Send Notification (Slack)
- **Type & Role:** Slack node; posts messages to Slack.
- **Configuration:**
  - Text: Uses the `notificationMessage` from input.
  - User/Channel: Configured with Slack User ID or Channel ID specified by `[YOUR_CYBERSECURITY_ALERT_CHANNEL_ID]`.
  - Uses Slack OAuth2 credentials.
- **Expressions/Variables:** Text expression:
  ```
  {{$json.notificationMessage}}
  ```
- **Input/Output:** Input from `Valid Mention?` (true branch); output to `End Workflow`.
- **Version Requirements:** Version 2.3.
- **Potential Failures:** Slack API authentication failure, invalid channel/user ID, rate limiting, network issues.
- **Sub-workflow:** None.

---

#### 2.5 Workflow Termination

- **Overview:** Marks the end of the workflow execution path.
- **Nodes Involved:** `End Workflow`

##### Node: End Workflow (NoOp)
- **Type & Role:** No-operation node; used to signify workflow completion.
- **Configuration:** No parameters.
- **Input/Output:** Receives input from either `Valid Mention?` (false branch) or `Send Notification`.
- **Version Requirements:** Version 1.
- **Potential Failures:** None.
- **Sub-workflow:** None.

---

### 3. Summary Table

| Node Name                  | Node Type           | Functional Role                      | Input Node(s)               | Output Node(s)              | Sticky Note                                                                                                                   |
|----------------------------|---------------------|------------------------------------|-----------------------------|-----------------------------|------------------------------------------------------------------------------------------------------------------------------|
| Schedule Trigger           | Schedule Trigger    | Initiates periodic workflow runs   | -                           | Monitor: Cybersecurity Keywords | ## Flow                                                                                                                      |
| Monitor: Cybersecurity Keywords | Twitter            | Searches X (Twitter) for keywords  | Schedule Trigger             | Format Notification          | ## Flow                                                                                                                      |
| Format Notification        | Code                | Formats tweet data into message    | Monitor: Cybersecurity Keywords | Valid Mention?               | ## Flow                                                                                                                      |
| Valid Mention?             | If                  | Filters out irrelevant mentions    | Format Notification          | Send Notification, End Workflow | ## Flow                                                                                                                      |
| Send Notification          | Slack               | Sends alerts to Slack channel      | Valid Mention?               | End Workflow                 | ## Flow                                                                                                                      |
| End Workflow              | NoOp                | Marks workflow completion          | Valid Mention?, Send Notification | -                           | ## Flow                                                                                                                      |
| Sticky Note                | Sticky Note         | Visual annotation                  | -                           | -                           | ## Flow                                                                                                                      |
| Sticky Note1               | Sticky Note         | Detailed documentation content     | -                           | -                           | # 🛡️ Simple Cybersecurity Brand/Vulnerability Mention Monitor 🚨 ... (full content provided in section 5 below)               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add and configure the `Schedule Trigger` node:**
   - Type: Schedule Trigger
   - Position: Top-left area
   - Parameters: Use default interval (every minute) or customize as needed.
   - No credentials required.

3. **Add a `Twitter` node named `Monitor: Cybersecurity Keywords`:**
   - Operation: Search
   - Search Text: Set to your cybersecurity keyword query, e.g.:
     ```
     ="[YourBrandName]" OR "CVE-2024-XXXX" OR "zeroday"
     ```
   - Limit: 10
   - Credentials: Create and select Twitter OAuth1 credential with your API keys.
   - Connect `Schedule Trigger` output to this node’s input.

4. **Add a `Code` node named `Format Notification`:**
   - Language: JavaScript
   - Code:
     ```js
     const tweet = $json.text;
     const user = $json.user.screen_name;
     const tweetUrl = `https://twitter.com/${user}/status/${$json.id_str}`;
     return [{
       json: {
         ...$json,
         notificationMessage: `🚨 New Cybersecurity Mention! 🚨\nUser: @${user}\nTweet: ${tweet}\nLink: ${tweetUrl}`
       }
     }];
     ```
   - Connect output of `Monitor: Cybersecurity Keywords` to this node’s input.

5. **Add an `If` node named `Valid Mention?`:**
   - Condition type: String
   - Condition: The field `notificationMessage` must NOT contain the string `"bot"` (case-sensitive).
   - Version: Use version 2.2 or higher.
   - Connect output of `Format Notification` to this node’s input.

6. **Add a `Slack` node named `Send Notification`:**
   - Operation: Send Message
   - Text: Use expression `{{$json.notificationMessage}}`
   - User/Channel: Select mode "ID" and input your Slack channel ID where alerts should post.
   - Credentials: Create and select Slack OAuth2 credential with correct scopes to post messages.
   - Connect `Valid Mention?` node’s `true` branch output to this node’s input.

7. **Add a `NoOp` node named `End Workflow`:**
   - No configuration needed.
   - Connect `Valid Mention?` node’s `false` branch output to `End Workflow`.
   - Connect output of `Send Notification` to `End Workflow`.

8. **Save and test the workflow:**
   - Run manually to verify messages appear in Slack.
   - Adjust Twitter search query and Slack channel as needed.
   - Activate the workflow to run continuously.

---

### 5. General Notes & Resources

| Note Content                 