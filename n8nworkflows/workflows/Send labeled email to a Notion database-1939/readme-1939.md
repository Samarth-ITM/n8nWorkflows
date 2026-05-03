Send labeled email to a Notion database

https://n8nworkflows.xyz/workflows/send-labeled-email-to-a-notion-database-1939


# Send labeled email to a Notion database

### 1. Workflow Overview

This workflow automates the process of syncing labeled Gmail emails into a Notion database as pages. It targets users who want to track specific emails by labeling them in Gmail, then have those emails reflected as tasks or notes inside Notion. The workflow includes the following logical blocks:

- **1.1 Scheduled Email Retrieval:** Periodically triggers to fetch all emails with a specific Gmail label.
- **1.2 Notion Database Lookup:** For each fetched email, checks if a corresponding Notion page already exists by matching the email thread ID.
- **1.3 Notion Page Creation:** If no page exists, creates a new Notion page with the email’s subject as title, a snippet of the body as content, and links back to the email.
- **1.4 Notion Page Update Monitoring:** Watches for updates to the Notion database pages, specifically looking for a "checked off" status to remove the Gmail label from the corresponding email.
- **1.5 Label Removal:** Removes the Gmail label from the email when the Notion page is marked complete.

This workflow ensures that only new emails with the designated label are added to Notion, and when the task is completed in Notion, the Gmail label is automatically removed to prevent duplication or reprocessing.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Email Retrieval

- **Overview:**  
  This block triggers the workflow every minute and retrieves all Gmail emails with the specified label.

- **Nodes Involved:**  
  - On schedule  
  - Derive last request time  
  - Get emails from label and last request time

- **Node Details:**

  - **On schedule**  
    - Type: Schedule Trigger  
    - Role: Triggers workflow every minute to start email retrieval.  
    - Parameters: Interval set to 1 minute.  
    - Input: None  
    - Output: Triggers next node with timestamp.  
    - Edge cases: Schedule delays, system downtime could affect timing.

  - **Derive last request time**  
    - Type: DateTime  
    - Role: Calculates a timestamp subtracting 1 minute from the current time to limit email query scope.  
    - Parameters: Subtract 1 minute from the current timestamp in input.  
    - Key expression: `={{ $json.timestamp }}` as base time, subtract 1 minute.  
    - Input: Timestamp from "On schedule"  
    - Output: Timestamp for filtering emails.  
    - Edge cases: Incorrect time zones or timestamp format may cause filtering issues.

  - **Get emails from label and last request time**  
    - Type: Gmail  
    - Role: Retrieves all emails with the designated Gmail label ("Notion") since the derived last request time.  
    - Parameters: Filter by label ID (configured as `Label_9178764513576607415`), get all matching emails.  
    - Credentials: Gmail OAuth2 required.  
    - Input: Timestamp (used internally for filtering, though not explicit in parameters here).  
    - Output: List of emails matching label.  
    - Edge cases: OAuth token expiration, Gmail API rate limits, label ID changes.

---

#### 2.2 Notion Database Lookup and Merge

- **Overview:**  
  For each email retrieved, checks if a Notion database page already exists for the email thread ID. Merges email data with Notion lookup results to determine if a new page should be created.

- **Nodes Involved:**  
  - Try get database page  
  - Merge  
  - If database page not found  
  - If found, do nothing

- **Node Details:**

  - **Try get database page**  
    - Type: Notion  
    - Role: Queries the Notion database to find pages where "Thread ID" matches the Gmail thread ID.  
    - Parameters: Filter condition on property "Thread ID" equals `={{ $json.id }}` (email thread ID).  
    - Credentials: Notion API required.  
    - Input: Emails from Gmail node.  
    - Output: Filtered Notion pages (if any).  
    - Edge cases: Notion API rate limits, incorrect database or property IDs, empty results.  
    - Always outputs data even if no pages found.

  - **Merge**  
    - Type: Merge  
    - Role: Combines email data with Notion query results, enriching email data with Notion page info if found.  
    - Parameters: Mode = Combine, Join mode = Enrich Input 1, merge by fields "id" (email) and "property_thread_id" (Notion).  
    - Input: Email data (input 1) and Notion page data (input 2).  
    - Output: Single enriched dataset per email.  
    - Edge cases: Mismatched IDs, empty second input leading to empty properties.

  - **If database page not found**  
    - Type: If  
    - Role: Checks if merged Notion property "property_thread_id" is empty, meaning no existing page found.  
    - Parameters: Condition checks if `={{ $json.property_thread_id }}` is empty.  
    - Input: Merged data.  
    - Output: If true, no page exists; if false, page exists.  
    - Edge cases: Property naming errors, empty or incorrect merge outputs.

  - **If found, do nothing**  
    - Type: NoOp  
    - Role: Placeholder to skip processing for emails that already have Notion pages.  
    - Input: If database page not found (false branch).  
    - Output: Ends branch.  
    - Edge cases: None.

---

#### 2.3 Notion Page Creation

- **Overview:**  
  For emails without existing Notion pages, creates a new page in the Notion database with email details.

- **Nodes Involved:**  
  - Find my email address  
  - Create database page

- **Node Details:**

  - **Find my email address**  
    - Type: HTTP Request  
    - Role: Retrieves the authenticated Gmail user's email address via Gmail API.  
    - Parameters: GET request to `https://gmail.googleapis.com/gmail/v1/users/me/profile`.  
    - Authentication: Gmail OAuth2 credentials.  
    - Input: From If database page not found (true branch).  
    - Output: JSON containing emailAddress used for constructing links.  
    - Edge cases: API errors, auth failures.

  - **Create database page**  
    - Type: Notion  
    - Role: Creates a new page in the Notion database.  
    - Parameters:  
      - Title: Email subject (`={{ $('If database page not found').item.json.Subject }}`)  
      - Content: Heading "Snippet" and snippet text from email body (`={{ $('If database page not found').item.json.snippet }}`)  
      - Link: "See more" link to Gmail email thread constructed with email address and message ID.  
      - Database ID: configured with target Notion database.  
      - Properties:  
        - Thread ID stored in rich text property "Thread ID"  
        - Email thread URL property "Email thread" set to Gmail email link  
      - Icon: Custom file icon URL shown.  
    - Credentials: Notion API required.  
    - Input: Email data enriched with user email.  
    - Output: Confirmation of page creation.  
    - Edge cases: Notion API errors, invalid database or property IDs, network issues.

---

#### 2.4 Notion Page Update Monitoring and Label Removal

- **Overview:**  
  Monitors Notion database pages for update events. When a page is marked complete (checked off), removes the Gmail label from the corresponding email.

- **Nodes Involved:**  
  - On updated database page  
  - If checked off  
  - Remove label from target email  
  - Not yet checked off, do nothing

- **Node Details:**

  - **On updated database page**  
    - Type: Notion Trigger  
    - Role: Polls Notion database every minute for updated pages.  
    - Parameters: Event type "pagedUpdatedInDatabase", poll every minute.  
    - Credentials: Notion API required.  
    - Input: None (trigger node).  
    - Output: Updated Notion page data.  
    - Edge cases: Polling delays, API rate limits.

  - **If checked off**  
    - Type: If  
    - Role: Checks if the "Complete" boolean property of the Notion page is true (checked off).  
    - Parameters: Condition comparing `$json.Complete` to `true`.  
    - Input: Updated page data.  
    - Output: True branch if checked off, false branch otherwise.  
    - Edge cases: Property naming or missing property.

  - **Remove label from target email**  
    - Type: Gmail  
    - Role: Removes the predefined Gmail label ("Notion") from the email with message ID matching Notion page Thread ID.  
    - Parameters: Label ID to remove, messageId from Notion page property `Thread ID`.  
    - Credentials: Gmail OAuth2 required.  
    - Input: True branch from "If checked off".  
    - Output: Confirmation of label removal.  
    - Edge cases: Gmail API errors, message not found, invalid label ID.

  - **Not yet checked off, do nothing**  
    - Type: NoOp  
    - Role: Placeholder when task is not marked complete, no action taken.  
    - Input: False branch from If checked off.  
    - Output: Ends branch.  
    - Edge cases: None.

---

### 3. Summary Table

| Node Name                      | Node Type               | Functional Role                                  | Input Node(s)                      | Output Node(s)                         | Sticky Note                                                                                                                                            |
|--------------------------------|-------------------------|-------------------------------------------------|----------------------------------|---------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------|
| Sticky Note                    | Sticky Note             | Documentation overview                           | None                             | None                                  | ## Send labeled email to a Notion database<br>This workflow sends the contents of an email to a Notion database. The email must be labeled with a specific label for the workflow to trigger. The email subject will be the title of the Notion page, and a snippet of the email body will be the content of the Notion page. The email link will be added to the Notion page as a property.<br><br>### How it works<br>On scheduled intervals, find all emails with a specific label. For each email, check if the email already exists in the Notion database. If it does not exist, create a new page in the Notion database, otherwise do nothing. When the task in the Notion database is checked off, the label will be removed from the email.<br><br>### Setup<br>This workflow requires that you set up a Notion database or use an existing one with at least the following fields:<br>- Title (title)<br>- Thread ID (text)<br>- Email thread (URL)<br><br>Additionally, create a label that will be used to trigger the workflow in Gmail. In this workflow, the label is called "Notion". |
| On schedule                   | Schedule Trigger        | Triggers workflow every minute                   | None                             | Derive last request time               |                                                                                                                                                         |
| Derive last request time      | DateTime                | Calculates timestamp 1 minute ago                | On schedule                     | Get emails from label and last request time |                                                                                                                                                         |
| Get emails from label and last request time | Gmail                   | Retrieves all emails with specific Gmail label  | Derive last request time          | Try get database page, Merge           |                                                                                                                                                         |
| Try get database page         | Notion                  | Queries Notion database for existing pages       | Get emails from label and last request time | Merge                                |                                                                                                                                                         |
| Merge                        | Merge                   | Combines email data with Notion page lookup      | Try get database page, Get emails from label and last request time | If database page not found            |                                                                                                                                                         |
| If database page not found    | If                      | Checks if Notion page exists for email thread    | Merge                           | Find my email address (true), If found, do nothing (false) |                                                                                                                                                         |
| Find my email address         | HTTP Request            | Retrieves Gmail user email address                | If database page not found (true) | Create database page                   |                                                                                                                                                         |
| Create database page          | Notion                  | Creates new Notion page with email content       | Find my email address           | None                                  |                                                                                                                                                         |
| If found, do nothing          | NoOp                    | Ends branch for emails with existing Notion pages | If database page not found (false) | None                                  |                                                                                                                                                         |
| On updated database page      | Notion Trigger          | Polls for updated pages in Notion database       | None                             | If checked off                        |                                                                                                                                                         |
| If checked off                | If                      | Checks if Notion page is marked complete         | On updated database page         | Remove label from target email (true), Not yet checked off, do nothing (false) |                                                                                                                                                         |
| Remove label from target email | Gmail                   | Removes Gmail label from email when completed    | If checked off (true)            | None                                  |                                                                                                                                                         |
| Not yet checked off, do nothing | NoOp                    | Ends branch when task not marked complete         | If checked off (false)           | None                                  |                                                                                                                                                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Trigger Node: "On schedule"**  
   - Node type: Schedule Trigger  
   - Set interval to trigger every 1 minute.

2. **Add "Derive last request time" Node**  
   - Node type: DateTime  
   - Configure to subtract 1 minute from the incoming timestamp `$json.timestamp`  
   - Output field name: `last_request_time` (used internally).

3. **Add "Get emails from label and last request time" Node**  
   - Node type: Gmail  
   - Operation: Get All emails  
   - Filter by label IDs: Use the Gmail label ID corresponding to your "Notion" label (e.g., `Label_9178764513576607415`).  
   - Credentials: Set Gmail OAuth2 credentials.  
   - Connect input from "Derive last request time".

4. **Add "Try get database page" Node**  
   - Node type: Notion  
   - Operation: Get All pages from database  
   - Filter: Property "Thread ID" (rich text) equals `={{ $json.id }}` (email thread ID)  
   - Credentials: Set Notion API credentials.  
   - Connect input from "Get emails from label and last request time".

5. **Add "Merge" Node**  
   - Node type: Merge  
   - Mode: Combine  
   - Join Mode: Enrich Input 1  
   - Merge by fields: `id` (from emails) and `property_thread_id` (from Notion pages)  
   - Connect input 1 from "Get emails from label and last request time"  
   - Connect input 2 from "Try get database page".

6. **Add "If database page not found" Node**  
   - Node type: If  
   - Condition: Check if `property_thread_id` is empty (string is empty)  
   - Connect input from "Merge".

7. **Add "Find my email address" Node**  
   - Node type: HTTP Request  
   - Method: GET  
   - URL: `https://gmail.googleapis.com/gmail/v1/users/me/profile`  
   - Authentication: Use Gmail OAuth2 credentials  
   - Connect true branch of "If database page not found".

8. **Add "Create database page" Node**  
   - Node type: Notion  
   - Operation: Create database page  
   - Database: Select target Notion database  
   - Title: Set to email subject, expression: `={{ $('If database page not found').item.json.Subject }}`  
   - Content blocks:  
     - Heading 3 block with text "Snippet"  
     - Text block with email snippet: `={{ $('If database page not found').item.json.snippet }}`  
     - RichText block with link "See more" pointing to Gmail email link constructed via expression: `=https://mail.google.com/mail/u/{{ $json.emailAddress }}/#all/{{ $('If database page not found').item.json.id }}`  
   - Properties:  
     - "Thread ID" (rich text): email thread ID `={{ $('If database page not found').item.json.id }}`  
     - "Email thread" (URL): Gmail email link as above  
   - Icon: Optional custom icon URL  
   - Credentials: Notion API credentials  
   - Connect input from "Find my email address".

9. **Add "If found, do nothing" Node**  
   - Node type: NoOp  
   - Connect false branch of "If database page not found".

10. **Add "On updated database page" Node**  
    - Node type: Notion Trigger  
    - Event: Page updated in database  
    - Poll interval: Every minute  
    - Database: Select same Notion database  
    - Credentials: Notion API credentials.

11. **Add "If checked off" Node**  
    - Node type: If  
    - Condition: Boolean property "Complete" equals true  
    - Connect input from "On updated database page".

12. **Add "Remove label from target email" Node**  
    - Node type: Gmail  
    - Operation: Remove Labels  
    - Label IDs: The Gmail label ID for "Notion"  
    - Message ID: Set to `={{ $json['Thread ID'] }}` from Notion page data  
    - Credentials: Gmail OAuth2 credentials  
    - Connect true branch of "If checked off".

13. **Add "Not yet checked off, do nothing" Node**  
    - Node type: NoOp  
    - Connect false branch of "If checked off".

14. **Wire all nodes according to the flow described:**  
    - "On schedule" → "Derive last request time" → "Get emails from label and last request time" → "Try get database page" and "Merge"  
    - "Merge" → "If database page not found" → True → "Find my email address" → "Create database page"  
    - "If database page not found" → False → "If found, do nothing"  
    - "On updated database page" → "If checked off" → True → "Remove label from target email"  
    - "If checked off" → False → "Not yet checked off, do nothing"

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                        | Context or Link                                                                                                                                                                 |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| This workflow requires a Notion database with at minimum these properties: Title (title), Thread ID (text), and Email thread (URL). Set up a Gmail label named "Notion" or equivalent, and replace label ID references accordingly. Ensure that the credentials for Gmail OAuth2 and Notion API are properly configured and updated in the workflow nodes. | Setup instructions in the workflow description. Credentials references: [Notion credentials](https://docs.n8n.io/integrations/builtin/credentials/notion/), [Google credentials](https://docs.n8n.io/integrations/builtin/credentials/google/) |
| Gmail label IDs are not the label names but system-generated IDs; to find the correct ID, use the Gmail API or n8n Gmail node list operation.                                                                                                                                                                                                       | Gmail label identification.                                                                                                                                                     |
| The Notion "Complete" property should be a checkbox property corresponding to task completion status.                                                                                                                                                                                                                                              | Important for label removal logic.                                                                                                                                               |

---

This document provides a comprehensive analysis and stepwise reconstruction guide for the "Send labeled email to a Notion database" workflow, enabling users and AI agents to understand, reproduce, and customize the process with confidence.