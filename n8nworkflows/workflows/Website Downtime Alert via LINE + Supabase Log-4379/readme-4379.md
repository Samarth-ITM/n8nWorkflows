Website Downtime Alert via LINE + Supabase Log

https://n8nworkflows.xyz/workflows/website-downtime-alert-via-line---supabase-log-4379


# Website Downtime Alert via LINE + Supabase Log

### 1. Workflow Overview

This workflow automates monitoring website uptime using UptimeRobot and issues alerts via the LINE messaging platform when one or more monitored websites are down or unstable. It also logs the downtime events into a Supabase database table for historical tracking and analysis. The workflow is designed to run on a schedule (every minute), check the current status of all monitored websites, filter those that are down or unstable, format a friendly and humorous alert message using OpenAI GPT-4o Mini, send this message to a LINE group, and store downed website details in Supabase.

The workflow is logically divided into the following blocks:

- **1.1 Scheduled Monitoring Trigger**: Initiates the workflow every minute.
- **1.2 Website Status Retrieval and Filtering**: Fetches monitor data from UptimeRobot and identifies problematic websites.
- **1.3 Conditional Branching on Downtime**: Checks if any monitored website is down or unstable.
- **1.4 Message Generation via LLM**: Uses an AI language model to create a customized alert message.
- **1.5 Message Preparation and Notification Delivery**: Escapes newlines and sends the message to LINE.
- **1.6 Logging Downtime Events**: Saves details of downed websites into Supabase.
- **1.7 Control Flow Utilities**: Includes batching and waiting nodes for orderly processing and rate limiting.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Monitoring Trigger

- **Overview:**  
Starts the workflow execution every minute to keep monitoring status updated continuously.

- **Nodes Involved:**  
  - Schedule Trigger

- **Node Details:**  
  - **Schedule Trigger**  
    - Type: `Schedule Trigger`  
    - Configuration: Interval set to every 1 minute.  
    - Inputs: None (entry point).  
    - Outputs: Connects to "Get Monitors".  
    - Edge Cases: Trigger misfires or downtime on n8n server could delay execution.  
    - Version: 1.2

#### 1.2 Website Status Retrieval and Filtering

- **Overview:**  
Fetches the current status of all monitored websites from UptimeRobot and filters out those with "down" or "unstable" statuses (codes 8 or 9), excluding a specific monitor ID (798534759). Constructs a summary message and flags if any site is down.

- **Nodes Involved:**  
  - Get Monitors  
  - Filter Down Monitors  
  - Loop Over Monitors

- **Node Details:**  
  - **Get Monitors**  
    - Type: `UptimeRobot` node  
    - Role: Retrieves all monitor data from UptimeRobot.  
    - Configuration: No filters applied; fetches full monitor list.  
    - Credentials: Uses a configured Uptime Robot API key.  
    - Input: From Schedule Trigger.  
    - Output: To Filter Down Monitors and Loop Over Monitors.  
    - Edge Cases: API rate limits, network errors, invalid credentials.  
    - Version: 1

  - **Filter Down Monitors**  
    - Type: `Code` (JavaScript) node  
    - Role: Processes the monitor list, builds a message listing down/unstable sites, and sets a flag `server_down` if any are found.  
    - Key Logic: Uses status codes with emoji mapping, excludes monitor ID 798534759.  
    - Input: From Get Monitors.  
    - Output: To If Down node.  
    - Edge Cases: Empty monitor list, missing fields, unexpected status codes.  
    - Version: 2

  - **Loop Over Monitors**  
    - Type: `SplitInBatches` node  
    - Role: Processes monitors individually for further filtering and logging.  
    - Configuration: Batch size dynamically set to the monitor's `id` value (likely a misconfiguration; should be a fixed number or undefined).  
    - Input: From Get Monitors.  
    - Output: To Save to Supabase and Filter Status 9 nodes.  
    - Edge Cases: Unusual batch size could cause errors or inefficiency.  
    - Version: 3

#### 1.3 Conditional Branching on Downtime

- **Overview:**  
Evaluates if there is any downtime detected (`server_down` flag set to `1`) to decide whether to proceed with alert generation.

- **Nodes Involved:**  
  - If Down

- **Node Details:**  
  - **If Down**  
    - Type: `If` node  
    - Role: Checks if `server_down` equals `"1"`.  
    - Input: From Filter Down Monitors.  
    - Output: True branch to LLM Message Format; false branch ends workflow (no action).  
    - Edge Cases: Type mismatches, missing `server_down` field.  
    - Version: 2.2

#### 1.4 Message Generation via LLM

- **Overview:**  
Formats the alert message by passing the raw text from the previous node to a LangChain LLM node, requesting a list of affected sites with a short humorous wish for the IT team. Then the message is fed into an OpenAI GPT-4o Mini model for refined generation.

- **Nodes Involved:**  
  - LLM Message Format  
  - OpenAI GPT-4o Mini

- **Node Details:**  
  - **LLM Message Format**  
    - Type: `LangChain ChainLlm` node  
    - Role: Defines a prompt that asks to list affected sites and add a humorous wish.  
    - Parameters: Uses expression to pass `$json.text` as input text.  
    - Input: From If Down.  
    - Output: To OpenAI GPT-4o Mini.  
    - Edge Cases: Failures in LangChain integration, prompt parsing errors.  
    - Version: 1.6

  - **OpenAI GPT-4o Mini**  
    - Type: `LangChain LmChatOpenAi` node  
    - Role: Sends prompt to OpenAI GPT-4o Mini model for natural language completion.  
    - Credentials: Uses OpenAI API key with correct scopes.  
    - Input: From LLM Message Format.  
    - Output: To Escape Newlines.  
    - Edge Cases: API rate limits, network errors, invalid API keys, model unavailability.  
    - Version: 1.2

#### 1.5 Message Preparation and Notification Delivery

- **Overview:**  
Escapes newlines in the generated message text to ensure proper JSON formatting, then sends the message as a text message to a LINE group chat via the LINE Messaging API. After sending, waits 30 minutes before allowing another alert.

- **Nodes Involved:**  
  - Escape Newlines  
  - Send to LINE  
  - Wait 30 Min

- **Node Details:**  
  - **Escape Newlines**  
    - Type: `Code` node  
    - Role: Replaces newline characters (`\n`) with escaped newlines (`\\n`) in the text.  
    - Input: From OpenAI GPT-4o Mini.  
    - Output: To Send to LINE.  
    - Edge Cases: Missing `text` field could cause error.  
    - Version: 2

  - **Send to LINE**  
    - Type: `HTTPRequest` node  
    - Role: Posts the escaped message to LINE Messaging API for pushing to a group.  
    - Configuration:  
      - Method: POST  
      - URL: `https://api.line.me/v2/bot/message/push`  
      - JSON body includes the target group ID and message text from expression.  
      - Headers include Authorization Bearer token (LINE Channel Token) and content type.  
    - Credentials: Requires LINE Messaging API channel token.  
    - Input: From Escape Newlines.  
    - Output: To Wait 30 Min.  
    - Edge Cases: Invalid token, network issues, API limits, malformed JSON.  
    - Version: 4.2

  - **Wait 30 Min**  
    - Type: `Wait` node  
    - Role: Pauses workflow for 30 minutes before the next iteration to avoid spamming notifications.  
    - Input: From Send to LINE.  
    - Output: None (ends the sequence).  
    - Edge Cases: Workflow stuck if interrupted during wait.  
    - Version: 1.1

#### 1.6 Logging Downtime Events

- **Overview:**  
Individually processes monitors that are down (status code 9) and saves relevant details (website name, status, uptime ID) into a Supabase table for record-keeping.

- **Nodes Involved:**  
  - Filter Status 9  
  - Save to Supabase

- **Node Details:**  
  - **Filter Status 9**  
    - Type: `Filter` node  
    - Role: Passes only monitors with status code 9 (down) for logging.  
    - Input: From Loop Over Monitors.  
    - Output: To Save to Supabase.  
    - Edge Cases: Missing status fields, incorrect data types.  
    - Version: 2.2

  - **Save to Supabase**  
    - Type: `Supabase` node  
    - Role: Inserts records of downed websites into the `synlora_uptime_down` table.  
    - Configuration: Maps fields `website`, `statue` (likely a typo for "status"), and `uptime_id` from monitor JSON.  
    - Credentials: Supabase API key with write access.  
    - Input: From Loop Over Monitors and Filter Status 9.  
    - Output: None.  
    - Edge Cases: API key expiration, table schema mismatch, network failures.  
    - Version: 1

#### 1.7 Control Flow Utilities

- **Overview:**  
Implements batch processing and flow control to handle multiple monitors efficiently.

- **Nodes Involved:**  
  - Loop Over Monitors (already described)  
  - Wait 30 Min (already described)

- **Node Details:**  
  - See node details above.

---

### 3. Summary Table

| Node Name           | Node Type                | Functional Role                            | Input Node(s)             | Output Node(s)              | Sticky Note                                   |
|---------------------|--------------------------|-------------------------------------------|---------------------------|-----------------------------|-----------------------------------------------|
| Schedule Trigger     | Schedule Trigger          | Trigger workflow every minute             | None                      | Get Monitors                |                                               |
| Get Monitors        | UptimeRobot               | Retrieve all monitor statuses              | Schedule Trigger           | Filter Down Monitors, Loop Over Monitors |                                               |
| Filter Down Monitors | Code                     | Filter monitors with down/unstable status | Get Monitors               | If Down                     |                                               |
| If Down             | If                       | Check if any monitor is down               | Filter Down Monitors       | LLM Message Format (true)    |                                               |
| LLM Message Format  | LangChain ChainLlm       | Format alert message prompt                 | If Down                   | OpenAI GPT-4o Mini           |                                               |
| OpenAI GPT-4o Mini  | LangChain LmChatOpenAi   | Generate natural language alert message    | LLM Message Format        | Escape Newlines             |                                               |
| Escape Newlines     | Code                     | Escape newlines in message text             | OpenAI GPT-4o Mini        | Send to LINE                |                                               |
| Send to LINE        | HTTP Request             | Send alert message to LINE group            | Escape Newlines           | Wait 30 Min                 |                                               |
| Wait 30 Min         | Wait                     | Pause to avoid notification spamming       | Send to LINE              | None                       |                                               |
| Loop Over Monitors  | SplitInBatches           | Process monitors individually               | Get Monitors              | Save to Supabase, Filter Status 9 |                                               |
| Filter Status 9     | Filter                   | Select monitors with status = 9 (down)     | Loop Over Monitors        | Save to Supabase            |                                               |
| Save to Supabase    | Supabase                 | Log down monitors to database               | Loop Over Monitors, Filter Status 9 | None                       |                                               |
| Sticky Note         | Sticky Note              | Visual comment: "Monitor Website Check when Down" | None                      | None                       | ## Monitor Website Check when Down ##          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Node**  
   - Add `Schedule Trigger` node.  
   - Set interval to run every 1 minute.

2. **Add UptimeRobot Node**  
   - Add `UptimeRobot` node named "Get Monitors".  
   - Set resource to "monitor" with no filters.  
   - Add your UptimeRobot API credentials.

3. **Add Code Node for Filtering Down Monitors**  
   - Add `Code` node named "Filter Down Monitors".  
   - Paste the JavaScript code that maps status codes and builds a message listing unstable/down monitors, excluding monitor ID 798534759.  
   - Output a JSON with `server_down` flag and `text` message.

4. **Add If Node for Downtime Check**  
   - Add `If` node named "If Down".  
   - Configure condition: `$json.server_down.toString() === "1"`.

5. **Add LangChain ChainLlm Node for Message Formatting**  
   - Add `LangChain ChainLlm` node named "LLM Message Format".  
   - Set prompt text to: `={{ $json.text }} List all affected sites and add a short funny wish for the IT team.`  
   - Set prompt type to "define".

6. **Add LangChain OpenAI Node**  
   - Add `LangChain LmChatOpenAi` node named "OpenAI GPT-4o Mini".  
   - Select model `gpt-4o-mini`.  
   - Add OpenAI API credentials.

7. **Add Code Node to Escape Newlines**  
   - Add `Code` node named "Escape Newlines".  
   - Paste JS code that replaces `\n` with `\\n` in the text.

8. **Add HTTP Request Node to Send to LINE**  
   - Add `HTTP Request` node named "Send to LINE".  
   - Configure POST request to `https://api.line.me/v2/bot/message/push`.  
   - Body (JSON):  
     ```json
     {
       "to": "<YOUR_LINE_GROUP_ID>",
       "messages": [
         {
           "type": "text",
           "text": "{{ $json.text }}"
         }
       ]
     }
     ```  
   - Headers:  
     - Authorization: `Bearer <YOUR_LINE_CHANNEL_TOKEN>`  
     - Content-Type: `application/json`  
   - Add LINE credentials or set tokens manually.

9. **Add Wait Node**  
   - Add `Wait` node named "Wait 30 Min".  
   - Set wait time to 30 minutes.

10. **Add SplitInBatches Node to Process Monitors Individually**  
    - Add `SplitInBatches` node named "Loop Over Monitors".  
    - Connect from "Get Monitors".  
    - Set batch size appropriately (e.g., 1 or fixed number—not the monitor id, which was likely a misconfiguration).

11. **Add Filter Node to Select Status 9 Monitors**  
    - Add `Filter` node named "Filter Status 9".  
    - Condition: `$json.status === '9'`.

12. **Add Supabase Node to Save Downtime Logs**  
    - Add `Supabase` node named "Save to Supabase".  
    - Configure table ID: `synlora_uptime_down`.  
    - Map fields:  
      - `website` = `{{ $json.friendly_name }}`  
      - `statue` = `{{ $json.status }}` (check typo, likely should be `status`)  
      - `uptime_id` = `{{ $json.id }}`  
    - Add Supabase credentials with write permission.

13. **Connect Nodes in this Order:**  
    - Schedule Trigger → Get Monitors  
    - Get Monitors → Filter Down Monitors  
    - Filter Down Monitors → If Down  
    - If Down (true) → LLM Message Format → OpenAI GPT-4o Mini → Escape Newlines → Send to LINE → Wait 30 Min  
    - Get Monitors → Loop Over Monitors → Filter Status 9 → Save to Supabase  
    - Loop Over Monitors → Save to Supabase (also directly from Loop Over Monitors, as per original)  

14. **Optional: Add Sticky Note**  
    - Add a sticky note with content: "## Monitor Website Check when Down ##" for documentation clarity.

---

### 5. General Notes & Resources

| Note Content                                                                                                             | Context or Link                            |
|--------------------------------------------------------------------------------------------------------------------------|--------------------------------------------|
| The monitor with ID `798534759` is explicitly excluded from downtime alerts in the filtering logic.                      | Monitor exclusion rationale                 |
| The workflow uses timezone "Asia/Bangkok" for timestamps in messages.                                                    | Timezone setting in Code node filter       |
| The Supabase field name `statue` appears to be a typo and likely should be `status`. Verify your database schema.         | Database field naming                       |
| LINE Messaging API requires a valid Group ID and Channel Token with appropriate permissions to send push messages.       | LINE API documentation: https://developers.line.biz/en/reference/messaging-api/ |
| OpenAI GPT-4o Mini model is used for message generation; ensure your OpenAI account has access to this model.             | OpenAI docs: https://platform.openai.com/docs/models/gpt-4o-mini |
| The workflow includes a 30-minute wait to avoid flooding the LINE group with repeated alerts.                            | Notification rate-limiting strategy         |

---

**Disclaimer:** The provided text is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and public.