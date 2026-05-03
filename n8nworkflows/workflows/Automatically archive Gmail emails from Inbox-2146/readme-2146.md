Automatically archive Gmail emails from Inbox

https://n8nworkflows.xyz/workflows/automatically-archive-gmail-emails-from-inbox-2146


# Automatically archive Gmail emails from Inbox

### 1. Workflow Overview

This workflow automates the archival of Gmail emails in the inbox received within the last day, except those that have been starred. It is designed to support an Inbox Zero strategy by removing unstarred emails from the inbox automatically on a schedule without manual intervention.

The workflow is logically divided into the following blocks:

- **1.1 Scheduled Trigger**: Initiates the workflow automatically at midnight on weekdays or customizable schedules.
- **1.2 Email Retrieval**: Fetches all Gmail threads with emails received in the last day that are currently labeled as Inbox.
- **1.3 Thread Inspection and Filtering**: Retrieves full details of each email thread and filters them based on whether any message in the thread is starred.
- **1.4 Archival Processing**: Archives (removes the Inbox label from) threads and individual messages accordingly:
  - Starred emails remain in the inbox.
  - Non-starred emails get archived.

Sticky notes guide setup and usage, emphasizing credential configuration and scheduling adjustments.


---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Trigger

- **Overview:**  
  This block triggers the workflow automatically at a configured time to ensure emails are processed regularly without manual start.

- **Nodes Involved:**  
  - At midnight every work day

- **Node Details:**  
  - **Node Name:** At midnight every work day  
  - **Type:** Schedule Trigger  
  - **Configuration:**  
    - Cron expression set to trigger at 00:00 (midnight) Monday through Friday (`0 0 * * 1-5`).  
    - This can be adjusted by the user for personal or other schedules.  
  - **Input:** None (trigger node)  
  - **Output:** Triggers "Get all emails in the last day" node  
  - **Edge Cases:**  
    - If the workflow is not enabled or credentials are invalid, the trigger will still fire but downstream nodes may fail.  
    - Cron expression misconfiguration can cause no or unexpected runs.  
  - **Sticky Note:**  
    - A note nearby advises to set schedules appropriately for personal or work email usage.

---

#### 2.2 Email Retrieval

- **Overview:**  
  Retrieves all Gmail threads from the inbox label where emails have been received in the last day.

- **Nodes Involved:**  
  - Get all emails in the last day  
  - Get the thread of each email

- **Node Details:**  

  - **Node Name:** Get all emails in the last day  
    - **Type:** Gmail node (resource: thread, operation: list)  
    - **Configuration:**  
      - Filter query: `label:inbox` (only inbox label)  
      - Received before: dynamically set to 24 hours ago (`{{$now.minus({days: 1})}}`)  
      - Return all threads matching criteria  
    - **Credentials:** Gmail OAuth2 (user must configure valid Gmail credentials)  
    - **Input:** Trigger from schedule  
    - **Output:** A list of thread IDs that match the query  
    - **Edge Cases:**  
      - API rate limits or authentication failure may cause retries or failure.  
      - If no emails match, downstream nodes receive empty data but do not fail.  
    - **Retry:** Enabled on fail.

  - **Node Name:** Get the thread of each email  
    - **Type:** Gmail node (resource: thread, operation: get)  
    - **Configuration:**  
      - For each thread ID from previous node, fetch full thread details (messages and metadata).  
    - **Credentials:** Same Gmail OAuth2 as above  
    - **Input:** Thread IDs from "Get all emails in the last day"  
    - **Output:** Detailed thread object including all messages and labels  
    - **Edge Cases:**  
      - Possible failures if thread is deleted between calls or API errors occur.  
    - **Retry:** Enabled on fail.

---

#### 2.3 Thread Inspection and Filtering

- **Overview:**  
  Filters threads to detect which contain starred messages and segregates them for different archival actions.

- **Nodes Involved:**  
  - Keep only starred emails in inbox  
  - for each message in the thread

- **Node Details:**  

  - **Node Name:** Keep only starred emails in inbox  
    - **Type:** Filter node  
    - **Configuration:**  
      - Boolean condition: Checks if the stringified messages array includes the string "STARRED".  
      - This detects if any message in the thread is starred.  
    - **Input:** Detailed thread data from "Get the thread of each email"  
    - **Output:**  
      - True branch: Threads containing starred messages  
      - False branch: Threads without starred messages (not explicitly used here but logically excluded)  
    - **Edge Cases:**  
      - If the message structure changes or "STARRED" label is not present as expected, filtering might fail or misclassify.  
      - This filter depends on the format of Gmail API response.  
    - **Sticky Note:**  
      - Explains the logic: "Keep starred emails in inbox.. Archive everything else!"

  - **Node Name:** for each message in the thread  
    - **Type:** Item Lists (splitter)  
    - **Configuration:**  
      - Splits the array of messages inside each thread into individual messages for processing.  
      - Field to split: "messages"  
    - **Input:** Threads from "Keep only starred emails in inbox"  
    - **Output:** Individual messages for downstream archival processing  
    - **Edge Cases:**  
      - If messages field missing or empty, node outputs empty or errors.  
      - Large threads with many messages may increase execution time.

---

#### 2.4 Archival Processing

- **Overview:**  
  Removes the Inbox label (archives) either at the thread level or for individual messages based on starring status.

- **Nodes Involved:**  
  - Archive thread (remove from inbox)  
  - Archive message (remove from inbox)

- **Node Details:**  

  - **Node Name:** Archive thread (remove from inbox)  
    - **Type:** Gmail node (resource: thread, operation: removeLabels)  
    - **Configuration:**  
      - Removes the Inbox label (`INBOX`) from the entire thread.  
      - Input threadId dynamically from filtered threads that contain starred messages.  
      - Note: According to the connections, this node is triggered on the true branch of the filter, which actually is "Keep only starred emails in inbox" — but logically it should archive non-starred emails. This suggests the filter logic or connections should be double-checked for correctness.  
    - **Credentials:** Gmail OAuth2  
    - **Input:** Threads filtered as starred  
    - **Output:** Success confirmation of label removal  
    - **Edge Cases:**  
      - API failure or label removal denial if thread is already archived or deleted.  
      - Possible race conditions if thread changes during processing.

  - **Node Name:** Archive message (remove from inbox)  
    - **Type:** Gmail node (resource: message, operation: removeLabels)  
    - **Configuration:**  
      - Removes Inbox label from each individual message.  
      - MessageId dynamically from the split messages.  
    - **Credentials:** Gmail OAuth2  
    - **Input:** Individual messages from "for each message in the thread"  
    - **Output:** Confirmation of label removal per message  
    - **Edge Cases:**  
      - Message might be already archived or deleted.  
      - Rate limits if processing very large number of messages.

---

### 3. Summary Table

| Node Name                     | Node Type             | Functional Role                         | Input Node(s)                 | Output Node(s)                             | Sticky Note                                                    |
|-------------------------------|----------------------|---------------------------------------|------------------------------|--------------------------------------------|----------------------------------------------------------------|
| Sticky Note3                  | Sticky Note          | Setup instructions                    |                              |                                            | ### 👨‍🎤 Setup 1. Add your Gmail creds                         |
| At midnight every work day    | Schedule Trigger     | Initiates workflow on schedule        |                              | Get all emails in the last day              | 👆 Set your schedule. I use this for work emails. For personal emails, I run this daily. |
| Get all emails in the last day | Gmail                | Fetch email threads from inbox last day | At midnight every work day    | Get the thread of each email                |                                                                |
| Get the thread of each email  | Gmail                | Retrieve full thread details           | Get all emails in the last day | Keep only starred emails in inbox           |                                                                |
| Keep only starred emails in inbox | Filter               | Filter threads containing starred emails | Get the thread of each email  | Archive thread (remove from inbox), for each message in the thread | ⭐ Keep starred emails in inbox.. Archive everything else!     |
| Archive thread  (remove from inbox) | Gmail                | Archive entire thread (remove Inbox label) | Keep only starred emails in inbox |                                            |                                                                |
| for each message in the thread | Item Lists           | Split thread messages for individual processing | Keep only starred emails in inbox | Archive message (remove from inbox)          |                                                                |
| Archive message (remove from inbox) | Gmail                | Archive individual messages (remove Inbox label) | for each message in the thread |                                            |                                                                |
| Sticky Note2                  | Sticky Note          | Scheduling advice                     |                              |                                            | 👆 Set your schedule. I use this for work emails. For personal emails, I run this daily. |
| Sticky Note                   | Sticky Note          | Explanation of filtering logic       |                              |                                            | ⭐ Keep starred emails in inbox.. Archive everything else!    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node**  
   - Name: "At midnight every work day"  
   - Type: Schedule Trigger  
   - Configure with cron expression: `0 0 * * 1-5` (midnight Monday to Friday)  
   - No credentials needed  

2. **Create a Gmail node to get all emails in the last day**  
   - Name: "Get all emails in the last day"  
   - Type: Gmail (Resource: Thread, Operation: List)  
   - Set filter query to: `label:inbox`  
   - Set "Received Before" to expression: `{{$now.minus({days: 1})}}`  
   - Enable "Return All" to true  
   - Attach Gmail OAuth2 credentials  
   - Connect output of Schedule Trigger to this node's input  

3. **Create a Gmail node to get the full thread details**  
   - Name: "Get the thread of each email"  
   - Type: Gmail (Resource: Thread, Operation: Get)  
   - Set Thread ID to expression: `{{$json.id}}` (from previous node output)  
   - Attach same Gmail OAuth2 credentials  
   - Connect output of "Get all emails in the last day" to this node's input  

4. **Add a Filter node to detect starred emails**  
   - Name: "Keep only starred emails in inbox"  
   - Type: Filter  
   - Condition: Boolean, Expression: `{{ JSON.stringify($json.messages).includes('STARRED') }}`  
   - Connect output of "Get the thread of each email" to this filter node  

5. **Create a Gmail node to archive threads**  
   - Name: "Archive thread  (remove from inbox)"  
   - Type: Gmail (Resource: Thread, Operation: removeLabels)  
   - Set Label IDs to: `INBOX`  
   - Set Thread ID to expression: `{{$json.id}}` (from filter node output)  
   - Attach Gmail OAuth2 credentials  
   - Connect the **true** output of the filter node to this node  

6. **Add an Item Lists node to split messages in each thread**  
   - Name: "for each message in the thread"  
   - Type: Item Lists  
   - Field to split out: `messages`  
   - Connect the **true** output of the filter node to this node as well  

7. **Create a Gmail node to archive individual messages**  
   - Name: "Archive message (remove from inbox)"  
   - Type: Gmail (Resource: Message, Operation: removeLabels)  
   - Set Label IDs to: `INBOX`  
   - Set Message ID to expression: `{{$json.id}}` (from the item list output)  
   - Attach Gmail OAuth2 credentials  
   - Connect output of "for each message in the thread" to this node  

8. **Add Sticky Notes for guidance**  
   - Place a sticky note near the trigger: "Setup: Add your Gmail creds"  
   - Place a sticky note near the schedule trigger: "Set your schedule. I use this for work emails. For personal emails, run daily."  
   - Place a sticky note near the filter node: "Keep starred emails in inbox.. Archive everything else!"  

9. **Configure Credentials**  
   - Add and authorize Gmail OAuth2 credentials in n8n, ensuring access to Gmail API with necessary scopes to read threads and modify labels.  

10. **Test the workflow**  
    - Manually trigger or wait for schedule to test.  
    - Verify emails received in the last day are archived if unstarred, starred emails remain in inbox.  

---

### 5. General Notes & Resources

| Note Content                                                                                         | Context or Link                                       |
|----------------------------------------------------------------------------------------------------|------------------------------------------------------|
| This workflow supports an Inbox Zero strategy by automatically archiving unstarred emails daily.   | Use case description                                  |
| Adjust the schedule according to personal or work email usage to optimize email management.        | Sticky notes in workflow                              |
| Gmail OAuth2 credentials must have appropriate scopes for reading and modifying email labels.      | Credential setup                                      |
| See n8n Gmail node documentation for advanced filtering and label management options.              | https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.gmail/ |
| If handling very large inboxes, consider API rate limits and split workload accordingly.           | General best practice for Gmail API usage            |

---

This document fully describes the workflow "Automatically archive Gmail emails from Inbox," enabling an advanced user or an AI agent to understand, reproduce, and maintain the workflow confidently.