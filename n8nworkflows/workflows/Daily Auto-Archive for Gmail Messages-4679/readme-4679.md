Daily Auto-Archive for Gmail Messages

https://n8nworkflows.xyz/workflows/daily-auto-archive-for-gmail-messages-4679


# Daily Auto-Archive for Gmail Messages

### 1. Workflow Overview

This workflow automates the daily archiving of Gmail inbox messages that are older than 24 hours. It is designed for users who want to keep their Gmail inbox clean by automatically removing the "INBOX" label from older emails, effectively archiving them. The process runs once every day at 4 AM and systematically processes all qualifying emails.

The workflow is logically divided into the following blocks:

- **1.1 Scheduled Trigger**: Initiates the workflow daily at a specified time.
- **1.2 Gmail Email Retrieval**: Fetches all emails from the Gmail inbox that were received before the last 24 hours.
- **1.3 Email Processing and Archiving**: Splits the batch of emails into individual messages and archives each by removing the "INBOX" label.

---

### 2. Block-by-Block Analysis

---

#### 1.1 Scheduled Trigger

- **Overview:**  
  This block triggers the workflow execution automatically every day at 4 AM.

- **Nodes Involved:**  
  - Daily Trigger at 4AM

- **Node Details:**

  - **Node Name:** Daily Trigger at 4AM  
    - **Type:** Schedule Trigger  
    - **Role:** Starts the workflow on a fixed daily schedule.  
    - **Configuration:**  
      - Set to trigger once daily at hour 4 (4 AM).  
    - **Expressions/Variables:** None.  
    - **Input:** None (trigger node).  
    - **Output:** Triggers the next node, "Fetch Gmail Inbox Emails".  
    - **Version:** 1.2  
    - **Potential Failures:**  
      - Scheduler misconfiguration (wrong timezone may cause unexpected trigger time).  
      - n8n instance downtime at trigger time may delay execution.  
    - **Sub-workflow:** None.

---

#### 1.2 Gmail Email Retrieval

- **Overview:**  
  Retrieves all emails in the Gmail inbox received before 24 hours ago, effectively targeting emails older than one day.

- **Nodes Involved:**  
  - Fetch Gmail Inbox Emails

- **Node Details:**

  - **Node Name:** Fetch Gmail Inbox Emails  
    - **Type:** Gmail Node  
    - **Role:** Queries Gmail API to fetch all inbox emails older than 24 hours.  
    - **Configuration:**  
      - Operation: Get All Messages.  
      - Filters:  
        - Label: INBOX (fetch only emails currently in inbox).  
        - Received Before: Date/time set to current time minus 24 hours (using JavaScript expression).  
      - Return All: true (fetch all matching emails without pagination).  
    - **Expressions:**  
      - `={{ new Date(Date.now() - 24 * 60 * 60 * 1000).toISOString() }}` dynamically calculates the timestamp 24 hours ago.  
    - **Input:** Triggered by "Daily Trigger at 4AM".  
    - **Output:** List of emails matching criteria passed to "Split Each Email".  
    - **Version:** 2.1  
    - **Credentials:** Uses Gmail OAuth2 credentials specific to the user’s Gmail account.  
    - **Potential Failures:**  
      - OAuth token expiration or authentication errors.  
      - API rate limits or quota exceeded by Gmail.  
      - Large inbox leading to long response times or timeouts.  
      - Expression evaluation errors (unlikely in this simple date calculation).  
    - **Sub-workflow:** None.

---

#### 1.3 Email Processing and Archiving

- **Overview:**  
  Processes the fetched emails individually and archives each by removing the "INBOX" label, effectively removing them from the inbox but keeping them in the user's Gmail account.

- **Nodes Involved:**  
  - Split Each Email  
  - Archive Message (Remove INBOX Label)

- **Node Details:**

  - **Node Name:** Split Each Email  
    - **Type:** Split Out  
    - **Role:** Splits the array of emails into individual email objects for separate processing.  
    - **Configuration:**  
      - Field to split out: `id` (each email's unique identifier).  
    - **Input:** Receives array of emails from "Fetch Gmail Inbox Emails".  
    - **Output:** Emits individual email objects, one per output item.  
    - **Version:** 1  
    - **Potential Failures:**  
      - Input data missing `id` field causing split failure.  
      - Empty input array results in no output and no further processing.  
    - **Sub-workflow:** None.

  - **Node Name:** Archive Message (Remove INBOX Label)  
    - **Type:** Gmail Node  
    - **Role:** Archives each individual email by removing the "INBOX" label.  
    - **Configuration:**  
      - Operation: Remove Labels.  
      - Labels to remove: ["INBOX"] (removes inbox label, effectively archiving).  
      - Message ID: Dynamically set using expression `={{ $json.id }}` to target current email.  
    - **Input:** Individual email id from "Split Each Email".  
    - **Output:** None (end of processing chain).  
    - **Version:** 2.1  
    - **Credentials:** Same Gmail OAuth2 credentials as "Fetch Gmail Inbox Emails".  
    - **Potential Failures:**  
      - Authentication or token expiration errors.  
      - API rate limits if too many emails processed at once.  
      - Invalid or missing message ID causing API errors.  
      - Network timeouts or Gmail API errors.  
    - **Sub-workflow:** None.

---

### 3. Summary Table

| Node Name                      | Node Type       | Functional Role                        | Input Node(s)          | Output Node(s)               | Sticky Note                        |
|-------------------------------|-----------------|-------------------------------------|-----------------------|-----------------------------|----------------------------------|
| Daily Trigger at 4AM           | Schedule Trigger| Initiates workflow daily at 4 AM    | None                  | Fetch Gmail Inbox Emails     |                                  |
| Fetch Gmail Inbox Emails       | Gmail           | Fetches all inbox emails older than 24h | Daily Trigger at 4AM  | Split Each Email             |                                  |
| Split Each Email               | Split Out       | Splits emails array into individual emails | Fetch Gmail Inbox Emails | Archive Message (Remove INBOX Label) |                                  |
| Archive Message (Remove INBOX Label) | Gmail    | Archives email by removing INBOX label | Split Each Email       | None                        |                                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Trigger Node:**
   - Add a **Schedule Trigger** node.
   - Configure it to run **daily** at **4 AM**.
   - Position it as the starting node.

2. **Add Gmail Fetch Node:**
   - Add a **Gmail** node.
   - Set the operation to **Get All Messages**.
   - Configure filters:
     - Label IDs: `["INBOX"]`
     - Received Before: `={{ new Date(Date.now() - 24 * 60 * 60 * 1000).toISOString() }}`  
       (This expression fetches emails older than 24 hours.)
   - Enable **Return All** to fetch all matching emails.
   - Connect the output of the Schedule Trigger node to this Gmail node.
   - Assign appropriate **Gmail OAuth2 credentials** for the account to be processed.

3. **Add Split Out Node:**
   - Add a **Split Out** node.
   - Set **Field to Split Out** to `id` (the unique identifier of each email).
   - Connect the output of the Gmail fetch node to this node.

4. **Add Gmail Archive Node:**
   - Add another **Gmail** node.
   - Set operation to **Remove Labels**.
   - Configure to remove label `INBOX`.
   - Set **Message ID** to `={{ $json.id }}` to dynamically target each email.
   - Connect the output of the Split Out node to this Gmail node.
   - Use the same Gmail OAuth2 credentials as the fetch node.

5. **Check and Save:**
   - Verify all connections are correct.
   - Save and activate the workflow.

---

### 5. General Notes & Resources

| Note Content                                                                                      | Context or Link                                            |
|-------------------------------------------------------------------------------------------------|------------------------------------------------------------|
| Gmail API has quota limits; processing a large number of emails daily may require quota monitoring. | Gmail API Quotas: https://developers.google.com/gmail/api/guides/quotas |
| Ensure Gmail OAuth2 credentials are authorized with sufficient permissions to read and modify labels. | Gmail OAuth2 Scopes: https://developers.google.com/gmail/api/auth/scopes |
| Timezone settings in n8n may affect the exact trigger time; verify instance timezone matches expected schedule. | n8n Timezone Settings: https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.schedule-trigger/ |
| Removing "INBOX" label archives the message but does not delete it; messages remain accessible via "All Mail". | Gmail Labels: https://support.google.com/mail/answer/6576    |

---

*Disclaimer: The text provided is extracted exclusively from an automated workflow created with n8n, a tool for integration and automation. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data processed is legal and public.*