Create a Pipedrive activity on Calendly event scheduled

https://n8nworkflows.xyz/workflows/create-a-pipedrive-activity-on-calendly-event-scheduled-1221


# Create a Pipedrive activity on Calendly event scheduled

### 1. Workflow Overview

This workflow automates follow-up activities triggered by Calendly event scheduling. When a meeting is scheduled through Calendly, it automatically creates a corresponding activity in Pipedrive to log the event. After the meeting ends, it waits 15 minutes and then sends a reminder message in Slack to the assigned interviewer, prompting them to document notes and insights from the meeting.

The workflow is organized into these logical blocks:

- **1.1 Event Reception:** Captures Calendly event scheduling in real-time.
- **1.2 Activity Creation:** Creates a matching activity in Pipedrive for sales or CRM tracking.
- **1.3 Feedback Timing Calculation:** Calculates the delay time (15 minutes after meeting end) to send a reminder.
- **1.4 Wait & Reminder Dispatch:** Waits until the calculated time, then sends a Slack reminder to the interviewer.

---

### 2. Block-by-Block Analysis

#### 2.1 Event Reception

- **Overview:**  
  This block listens for newly scheduled meetings via Calendly and triggers the workflow upon invitee creation.

- **Nodes Involved:**  
  - Calendly Trigger

- **Node Details:**  
  - **Name:** Calendly Trigger  
  - **Type:** Calendly Trigger node  
  - **Role:** Listens to Calendly webhook events; starts workflow on new invitee creation.  
  - **Configuration:**  
    - Trigger event set to `"invitee.created"` which fires when a meeting is scheduled.  
    - Uses stored Calendly API credentials for authentication.  
  - **Input:** None (webhook trigger).  
  - **Output:** Emits event payload containing detailed meeting and invitee information.  
  - **Edge Cases / Failures:**  
    - Webhook registration failure or loss of webhook subscription may cause missed triggers.  
    - Invalid or expired Calendly API credentials lead to authentication errors.  
    - Payload schema changes from Calendly could break downstream processing.  
  - **Sub-workflow:** None.

---

#### 2.2 Activity Creation

- **Overview:**  
  Creates a new "call" activity in Pipedrive corresponding to the scheduled Calendly event, logging meeting details.

- **Nodes Involved:**  
  - Pipedrive

- **Node Details:**  
  - **Name:** Pipedrive  
  - **Type:** Pipedrive node  
  - **Role:** Creates an activity record in Pipedrive CRM.  
  - **Configuration:**  
    - Resource: `activity`  
    - Type: `"call"`  
    - Subject dynamically generated from Calendly event details using expressions:  
      `{{$json["payload"]["event_type"]["name"]}} with {{$json["payload"]["invitee"]["name"]}} on {{$json["payload"]["event"]["invitee_start_time"]}}`  
    - No additional fields set.  
    - Uses Pipedrive API credentials.  
  - **Input:** Data from Calendly Trigger node via JSON payload.  
  - **Output:** Created activity data returned from Pipedrive API.  
  - **Edge Cases / Failures:**  
    - Authentication failure due to invalid Pipedrive credentials.  
    - API rate limits or downtime causing creation failure.  
    - Missing or malformed event data causing subject expression errors.  
  - **Sub-workflow:** None.

---

#### 2.3 Feedback Timing Calculation

- **Overview:**  
  Computes the exact timestamp 15 minutes after the meeting end time to schedule the Slack reminder.

- **Nodes Involved:**  
  - Date & Time

- **Node Details:**  
  - **Name:** Date & Time  
  - **Type:** Date & Time node  
  - **Role:** Calculates a future datetime based on meeting end time plus 15 minutes.  
  - **Configuration:**  
    - Input datetime taken from `{{$json["payload"]["event"]["end_time"]}}` (meeting end time).  
    - Action: Calculate new datetime by adding 15 minutes.  
    - Result stored in a new property `feedback_time`.  
  - **Input:** Calendly Trigger output JSON payload.  
  - **Output:** JSON augmented with `feedback_time` timestamp.  
  - **Edge Cases / Failures:**  
    - Invalid or missing `end_time` data causing calculation failure.  
    - Timezone mismatches potentially causing incorrect scheduling.  
  - **Sub-workflow:** None.

---

#### 2.4 Wait & Reminder Dispatch

- **Overview:**  
  Waits until 15 minutes after meeting end, then sends a Slack message reminder to the assigned interviewer.

- **Nodes Involved:**  
  - Wait  
  - Slack

- **Node Details:**  

  - **Wait Node:**  
    - **Type:** Wait node  
    - **Role:** Pauses workflow execution until the calculated `feedback_time`.  
    - **Configuration:**  
      - Resume mode: `specificTime`  
      - DateTime expression: `{{$json["feedback_time"]}}`  
    - Input: Output from Date & Time node (with `feedback_time` property).  
    - Output: Passes message to Slack node after wait.  
    - Edge Cases:  
      - If `feedback_time` is in the past or invalid, wait duration may be zero or invalid.  
      - Workflow execution limits or crashes during wait.  

  - **Slack Node:**  
    - **Type:** Slack node  
    - **Role:** Sends a message reminder in Slack channel to the interviewer.  
    - **Configuration:**  
      - Channel: `salesteam`  
      - Text message dynamically constructed using expressions:  
        ```
        {{$json["payload"]["event"]["assigned_to"][0]}}, today you had a {{$json["payload"]["event_type"]["name"]}} {{$json["payload"]["event_type"]["kind"]}} meeting with {{$json["payload"]["invitee"]["name"]}}. Please write your notes from the call here [link] and mark this message with ✅ when you're done.
        ```  
      - No blocks or attachments used.  
      - Uses Slack API credentials.  
    - Input: Data passed from Wait node.  
    - Output: Slack API response.  
    - Edge Cases / Failures:  
      - Invalid or missing Slack API credentials.  
      - Channel not found or access denied.  
      - Mentioning user with invalid ID or missing `assigned_to` data.  
      - Network or API rate limit issues.  
    - Sub-workflow: None.

---

### 3. Summary Table

| Node Name       | Node Type            | Functional Role              | Input Node(s)      | Output Node(s) | Sticky Note                                                                                     |
|-----------------|----------------------|-----------------------------|--------------------|----------------|------------------------------------------------------------------------------------------------|
| Calendly Trigger| Calendly Trigger     | Receive Calendly event webhook | None               | Date & Time, Pipedrive |                                                                                                |
| Pipedrive       | Pipedrive            | Create Pipedrive activity    | Calendly Trigger   | None           |                                                                                                |
| Date & Time     | Date & Time          | Calculate reminder time      | Calendly Trigger   | Wait           |                                                                                                |
| Wait            | Wait                 | Wait until reminder time     | Date & Time        | Slack          |                                                                                                |
| Slack           | Slack                | Send reminder message        | Wait               | None           |                                                                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Calendly Trigger Node**
   - Type: Calendly Trigger  
   - Configuration:  
     - Set event to `invitee.created` (fires when a new meeting is scheduled).  
     - Connect your Calendly API credentials (OAuth or API key).  
   - Position: Start node, no input.

2. **Create Pipedrive Node**
   - Type: Pipedrive  
   - Configuration:  
     - Resource: `activity`  
     - Operation: `create` (default)  
     - Type: `call`  
     - Subject: Use expression to concatenate:  
       ```
       {{$json["payload"]["event_type"]["name"]}} with {{$json["payload"]["invitee"]["name"]}} on {{$json["payload"]["event"]["invitee_start_time"]}}
       ```  
     - Connect your Pipedrive API credentials.  
   - Connect input from Calendly Trigger node.

3. **Create Date & Time Node**
   - Type: Date & Time  
   - Configuration:  
     - Action: Calculate  
     - Value: Expression:  
       ```
       {{$json["payload"]["event"]["end_time"]}}
       ```  
     - Duration: 15  
     - Time Unit: Minutes  
     - Store result in property: `feedback_time`  
   - Connect input from Calendly Trigger node.

4. **Create Wait Node**
   - Type: Wait  
   - Configuration:  
     - Resume: `specificTime`  
     - DateTime: Expression:  
       ```
       {{$json["feedback_time"]}}
       ```  
   - Connect input from Date & Time node.

5. **Create Slack Node**
   - Type: Slack  
   - Configuration:  
     - Channel: `salesteam` (or your desired Slack channel)  
     - Text: Use expression:  
       ```
       {{$json["payload"]["event"]["assigned_to"][0]}}, today you had a {{$json["payload"]["event_type"]["name"]}} {{$json["payload"]["event_type"]["kind"]}} meeting with {{$json["payload"]["invitee"]["name"]}}. Please write your notes from the call here [link] and mark this message with ✅ when you're done.
       ```  
     - Connect your Slack API credentials.  
   - Connect input from Wait node.

6. **Connect Workflow**
   - From Calendly Trigger node, connect outputs to both Pipedrive and Date & Time nodes.
   - Connect Date & Time node output to Wait node.
   - Connect Wait node output to Slack node.

7. **Credential Setup**
   - Calendly API: OAuth2 or API key with permissions to read invitee data.  
   - Pipedrive API: API token with write access to activities.  
   - Slack API: OAuth2 token or bot token with permission to post messages in target channel.

---

### 5. General Notes & Resources

| Note Content                                                                                             | Context or Link                   |
|---------------------------------------------------------------------------------------------------------|---------------------------------|
| Reminder message includes a placeholder `[link]` that should be replaced with an actual note-taking URL. | Slack node message configuration |
| Ensure timezones in Calendly event payload and n8n server match to avoid incorrect scheduling.           | Date & Time node usage           |
| Slack user mention uses `assigned_to[0]` which assumes at least one assignee; verify data presence.       | Slack message dynamic expression |
| Pipedrive "call" activity type is used; adjust if your CRM setup uses different activity types.          | Pipedrive node configuration    |
| For production, monitor webhook reliability and handle possible webhook unsubscriptions from Calendly.   | Calendly Trigger node            |

---

This documentation fully describes the workflow’s functionality, node configuration, and reproduction steps, enabling users or automation agents to understand, maintain, and extend the workflow efficiently.