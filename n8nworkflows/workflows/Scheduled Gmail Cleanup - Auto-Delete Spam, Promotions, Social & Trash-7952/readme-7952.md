Scheduled Gmail Cleanup - Auto-Delete Spam, Promotions, Social & Trash

https://n8nworkflows.xyz/workflows/scheduled-gmail-cleanup---auto-delete-spam--promotions--social---trash-7952


# Scheduled Gmail Cleanup - Auto-Delete Spam, Promotions, Social & Trash

### 1. Workflow Overview

This workflow automates the cleanup of unwanted emails in Gmail to maintain inbox hygiene and free up storage space. It is designed for users who want to automatically delete emails from Gmail’s Spam, Promotions, Social, and Trash folders on a scheduled basis, minimizing manual email management effort.

The workflow consists of the following logical blocks:

- **1.1 Schedule Trigger**: Initiates the workflow execution automatically on a daily schedule (customizable).
- **1.2 Fetch Emails by Category**: Uses Gmail nodes to retrieve all emails from Spam, Promotions, Social categories, and Trash folder, optionally filtering to recent emails.
- **1.3 Delete Emails**: Deletes the fetched emails permanently from Gmail.

This modular structure allows easy customization by adding or removing email category nodes or adjusting filters and deletion behavior.

---

### 2. Block-by-Block Analysis

#### Block 1.1: Schedule Trigger

- **Overview**:  
  Starts the workflow automatically at a configured time interval (default: daily at midnight). This ensures cleanup runs regularly without user intervention.

- **Nodes Involved**:  
  - Trigger every day at midnight

- **Node Details**:  

  - **Trigger every day at midnight**  
    - Type: Schedule Trigger  
    - Configuration: Runs once daily at midnight by default; interval can be customized to other times or frequencies.  
    - Expressions/Variables: None (time set directly in parameters).  
    - Input: None (trigger node).  
    - Output: Initiates the next nodes fetching emails.  
    - Edge cases:  
      - If the node is disabled or schedule misconfigured, the workflow will not run automatically.  
      - Timezone settings can affect execution time.  
    - Version-specific: Uses n8n version 1.2 or newer for schedule node capabilities.

---

#### Block 1.2: Fetch Emails by Category

- **Overview**:  
  Retrieves emails from Gmail’s Spam, Promotions, Social categories, and Trash folder. Each category is fetched separately to allow selective management and filtering. By default, it fetches emails received in the last 5 days for Social, Promotions, and Trash categories; Spam fetch has no date filter.

- **Nodes Involved**:  
  - Get all SPAM Mails  
  - Get all Social Mails  
  - Get all Trash Mails  
  - Get all Promotional Mails  

- **Node Details**:  

  - **Get all SPAM Mails**  
    - Type: Gmail node  
    - Role: Fetch all emails labeled as Spam  
    - Configuration:  
      - Label filter: "SPAM"  
      - Operation: "getAll"  
      - Return all emails matching filter (returnAll=true)  
      - No date filter applied  
    - Credentials: Gmail OAuth2 account connected.  
    - Input: Trigger node output  
    - Output: List of Spam emails to delete  
    - Edge cases:  
      - Large number of spam emails can slow execution.  
      - Gmail API quota limits might apply.  
      - Authentication errors if token expires.  

  - **Get all Social Mails**  
    - Type: Gmail node  
    - Role: Fetch emails from Social category (social network notifications)  
    - Configuration:  
      - Label filter: "CATEGORY_SOCIAL"  
      - Operation: "getAll"  
      - Return all emails received after 5 days ago (`={{ $today.minus({ days: 5 }) }}`)  
      - returnAll=true  
    - Credentials: Shared Gmail OAuth2 account.  
    - Input: Trigger node output  
    - Output: List of Social emails to delete  
    - Edge cases:  
      - Date filter can be adjusted; misconfiguration could exclude relevant emails.  
      - Gmail API quota and auth limits apply.  

  - **Get all Trash Mails**  
    - Type: Gmail node  
    - Role: Fetch emails currently in Trash folder  
    - Configuration:  
      - Label filter: "TRASH"  
      - Operation: "getAll"  
      - Received after 5 days ago (`={{ $today.minus({ days: 5 }) }}`)  
      - returnAll=true  
    - Credentials: Gmail OAuth2 account.  
    - Input: Trigger node output  
    - Output: List of Trash emails for permanent deletion  
    - Edge cases:  
      - Trash folder emails can be deleted automatically by Gmail after 30 days; this node allows earlier deletion.  
      - Date filter adjustable to avoid premature deletion.  

  - **Get all Promotional Mails**  
    - Type: Gmail node  
    - Role: Fetch emails from Promotions category (marketing, offers)  
    - Configuration:  
      - Label filter: "CATEGORY_PROMOTIONS"  
      - Operation: "getAll"  
      - Received after 5 days ago (`={{ $today.minus({ days: 5 }) }}`)  
      - returnAll=true  
    - Credentials: Gmail OAuth2 account.  
    - Input: Trigger node output  
    - Output: List of promotional emails to delete  
    - Edge cases:  
      - Potential accidental deletion of important promotional emails if filters not adjusted.  
      - API or auth failures.  

---

#### Block 1.3: Delete Emails

- **Overview**:  
  Permanently deletes all emails fetched in the previous nodes. This step frees up Gmail storage and clears clutter. The deletion is irreversible by default but can be modified to move emails to Trash instead.

- **Nodes Involved**:  
  - Delete Mails

- **Node Details**:  

  - **Delete Mails**  
    - Type: Gmail node  
    - Role: Delete emails by message ID  
    - Configuration:  
      - Operation: "delete"  
      - Message ID is dynamically set from each input email’s `id` field (`={{ $json.id }}`)  
    - Credentials: Same Gmail OAuth2 account used in fetching nodes.  
    - Input: Connected from each "Get all ..." node outputs (parallel).  
    - Output: None (deletion operation)  
    - Edge cases:  
      - If message ID is missing or malformed, deletion will fail.  
      - API rate limits or quota exceeded errors possible.  
      - Authentication tokens must be valid; expired tokens cause failure.  
      - Deletion is permanent; accidental deletion risks exist. Backup or confirmation not included.  

---

### 3. Summary Table

| Node Name            | Node Type         | Functional Role           | Input Node(s)           | Output Node(s)        | Sticky Note                                                                                              |
|----------------------|-------------------|--------------------------|------------------------|-----------------------|--------------------------------------------------------------------------------------------------------|
| Sticky Note          | Sticky Note       | Documentation note       | -                      | -                     | ## Scheduled Gmail Cleanup - Auto-Delete Spam, Promotions, Social & Trash ... (full content in note)   |
| Sticky Note1         | Sticky Note       | Documentation note       | -                      | -                     | ## Schedule Trigger :point_up: ... (explains schedule trigger customization)                           |
| Get all SPAM Mails   | Gmail             | Fetch Spam emails        | Trigger every day at midnight | Delete Mails         | ## :point_down: Fetch Spam Emails ... (details about Spam email fetching)                              |
| Get all Social Mails | Gmail             | Fetch Social emails      | Trigger every day at midnight | Delete Mails         | ## :point_down: Fetch Social Emails ... (details about Social email fetching)                          |
| Get all Trash Mails  | Gmail             | Fetch Trash emails       | Trigger every day at midnight | Delete Mails         | ## :point_down: Fetch Trash Emails ... (details about Trash email fetching)                            |
| Get all Promotional Mails | Gmail         | Fetch Promotions emails  | Trigger every day at midnight | Delete Mails         | ## :point_down: Fetch Promotions Emails ... (details about Promotions email fetching)                  |
| Sticky Note2         | Sticky Note       | Documentation note       | -                      | -                     | ## :point_down: Fetch Social Emails ... (explains Social mail node)                                    |
| Sticky Note3         | Sticky Note       | Documentation note       | -                      | -                     | ## :point_down: Fetch Spam Emails ... (explains Spam mail node)                                        |
| Sticky Note4         | Sticky Note       | Documentation note       | -                      | -                     | ## :point_down: Fetch Trash Emails ... (explains Trash mail node)                                      |
| Sticky Note5         | Sticky Note       | Documentation note       | -                      | -                     | ## :point_down: Fetch Promotions Emails ... (explains Promotions mail node)                            |
| Delete Mails         | Gmail             | Delete fetched emails    | Get all SPAM Mails, Get all Social Mails, Get all Trash Mails, Get all Promotional Mails | - | ## :point_up: Delete Emails ... (explains deletion behavior and options)                              |
| Sticky Note6         | Sticky Note       | Documentation note       | -                      | -                     | ## :point_up: Delete Emails ... (explains deletion node usage)                                        |
| Trigger every day at midnight | Schedule Trigger | Initiates workflow      | -                      | Get all SPAM Mails, Get all Social Mails, Get all Trash Mails, Get all Promotional Mails | ## Schedule Trigger :point_up: ... (trigger customization) |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a Schedule Trigger node**  
   - Name: `Trigger every day at midnight`  
   - Type: Schedule Trigger  
   - Parameters: Set interval to daily at midnight (00:00). Adjust timezone if needed.  
   - No credentials needed.

3. **Add Gmail credential**  
   - Configure a Gmail OAuth2 credential with access to Gmail API scopes for reading and deleting emails.

4. **Create Gmail nodes to fetch emails per category:**  

   - **Get all SPAM Mails**  
     - Type: Gmail  
     - Name: `Get all SPAM Mails`  
     - Operation: `getAll`  
     - Filter: labelIds = `[ "SPAM" ]`  
     - Return All: true  
     - Credential: Use Gmail OAuth2 credential  
     - Input: Connect from Schedule Trigger output  

   - **Get all Social Mails**  
     - Type: Gmail  
     - Name: `Get all Social Mails`  
     - Operation: `getAll`  
     - Filter: labelIds = `[ "CATEGORY_SOCIAL" ]`  
     - Filter date: receivedAfter = `={{ $today.minus({ days: 5 }) }}` (fetch emails from last 5 days)  
     - Return All: true  
     - Credential: Gmail OAuth2  
     - Input: Connect from Schedule Trigger output  

   - **Get all Trash Mails**  
     - Type: Gmail  
     - Name: `Get all Trash Mails`  
     - Operation: `getAll`  
     - Filter: labelIds = `[ "TRASH" ]`  
     - Filter date: receivedAfter = `={{ $today.minus({ days: 5 }) }}`  
     - Return All: true  
     - Credential: Gmail OAuth2  
     - Input: Connect from Schedule Trigger output  

   - **Get all Promotional Mails**  
     - Type: Gmail  
     - Name: `Get all Promotional Mails`  
     - Operation: `getAll`  
     - Filter: labelIds = `[ "CATEGORY_PROMOTIONS" ]`  
     - Filter date: receivedAfter = `={{ $today.minus({ days: 5 }) }}`  
     - Return All: true  
     - Credential: Gmail OAuth2  
     - Input: Connect from Schedule Trigger output  

5. **Add a Gmail node to delete emails:**  

   - Type: Gmail  
   - Name: `Delete Mails`  
   - Operation: `delete`  
   - Message ID parameter: `={{ $json.id }}` (dynamically delete each email by its ID)  
   - Credential: Same Gmail OAuth2 credential  
   - Input: Connect outputs from all four "Get all ..." nodes to this node’s input (parallel connections).

6. **Add Sticky Notes (optional) for documentation:**  
   - Add notes explaining the Schedule trigger, each fetch node’s purpose and filters, and the Delete node’s behavior.

7. **Activate the workflow** to run automatically on schedule.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                          | Context or Link                                                                                                                                     |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------|
| Managing Gmail manually can be tedious and time-consuming. This workflow automates cleaning Spam, Promotions, Social, and Trash emails to keep your inbox cleaner with zero daily effort. Customize schedule frequency and filters to tailor deletion behavior.                                                                                                                 | Main Sticky Note content at workflow start.                                                                                                       |
| Schedule Trigger node is easily customizable to run at different intervals such as hourly, daily, or weekly.                                                                                                                                                                                                                          | Sticky Note1 content explaining schedule customization.                                                                                           |
| Gmail API quotas and rate limits apply. Ensure your Gmail OAuth2 credentials have appropriate scopes and are refreshed to avoid auth errors. Consider API limitations when scheduling frequency or deleting large volumes of emails.                                                                                                   | General best practice note for Gmail API usage.                                                                                                  |
| Deletion is permanent by default; modify the Delete Mails node to move emails to Trash instead for safer cleanup if desired.                                                                                                                                                                                                           | Sticky Note6 content.                                                                                                                             |
| Gmail automatically deletes Trash items after about 30 days; this workflow can accelerate cleanup by deleting Trash emails after 5 days or any configured timeframe.                                                                                                                                                                 | Block 1.2 Trash fetch node detail.                                                                                                               |
| Expressions like `={{ $today.minus({ days: 5 }) }}` are used to filter emails newer than 5 days; adjust as needed to prevent deleting recent important emails.                                                                                                                                                                       | Explanation of date filters used in fetch nodes.                                                                                                |

---

**Disclaimer:**  
The text above is based exclusively on an automated workflow built with n8n, an integration and automation tool. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All data processed are legal and public.