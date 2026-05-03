Email Notifications to Telegram using Temporary HTML URL Hosting via GitHub Gist

https://n8nworkflows.xyz/workflows/email-notifications-to-telegram-using-temporary-html-url-hosting-via-github-gist-2080


# Email Notifications to Telegram using Temporary HTML URL Hosting via GitHub Gist

### 1. Workflow Overview

This workflow automates email notifications by detecting new incoming emails via IMAP, then sending immediate notifications to a Telegram chat with a secure, temporary HTML preview of the email. The email content is hosted temporarily as a private GitHub Gist, enabling users to view the email in a browser without exposing sensitive content directly in Telegram. After a configurable delay, the hosted HTML page and the Telegram notification are both deleted to maintain privacy and security.

Logical blocks:

- **1.1 Email Reception and Processing:** Detect new emails via IMAP, extract email content and metadata.
- **1.2 Temporary HTML Hosting:** Create a private GitHub Gist containing the email's HTML content to serve as a temporary preview page.
- **1.3 Telegram Notification:** Send a Telegram message with a link to the hosted HTML page and inline button for convenient access.
- **1.4 Timed Cleanup:** After a delay (3 hours by default), delete the GitHub Gist and remove the Telegram notification to ensure content is available only temporarily.

---

### 2. Block-by-Block Analysis

#### 2.1 Email Reception and Processing

**Overview:**  
This block triggers the workflow on receiving new emails, fetching the full resolved email including HTML content and metadata such as sender and recipient addresses.

**Nodes Involved:**  
- Email Trigger (IMAP)

**Node Details:**  

- **Email Trigger (IMAP)**  
  - Type: Email Trigger (IMAP)  
  - Role: Monitors the email inbox for unread (UNSEEN) emails and triggers the workflow when a new email arrives.  
  - Configuration:  
    - Format: Resolved (fetches parsed email content and metadata)  
    - Custom email config: `["UNSEEN"]` to process only new unread emails.  
    - Force reconnect every 60 seconds to maintain connection stability.  
  - Key Expressions/Variables: Outputs full email JSON including `from`, `to`, `html`, and `date`.  
  - Input: None (trigger node)  
  - Output: Passes email data to the Github Gist creation node.  
  - Potential Failures: IMAP connection/authentication errors, network timeouts, malformed emails.  
  - Version: n8n version supporting Email Trigger (IMAP) v2.

---

#### 2.2 Temporary HTML Hosting

**Overview:**  
This block creates a private GitHub Gist containing the email’s HTML content to provide a secure, temporary web page for previewing the email.

**Nodes Involved:**  
- Github Gist (Create)

**Node Details:**  

- **Github Gist**  
  - Type: HTTP Request (POST to GitHub Gist API)  
  - Role: Creates a private Gist with a single file `email.html` containing the raw HTML of the email.  
  - Configuration:  
    - URL: `https://api.github.com/gists` (GitHub Gist creation endpoint)  
    - Method: POST  
    - Authentication: Predefined GitHub API credential (token with gist creation permission)  
    - Headers: Accept header set to `application/vnd.github+json`  
    - Body (JSON):  
      - `description`: Includes email date, sender, and recipient addresses dynamically extracted from the email JSON.  
      - `public`: false (private gist)  
      - `files`: A single file named `email.html` containing the HTML content of the email.  
    - Expressions used to extract and sanitize HTML and email addresses for the gist content and description.  
  - Input: Email JSON from Email Trigger.  
  - Output: GitHub response including the `id` of the created gist and file metadata, passed on to the Telegram notification node.  
  - Failure Types: GitHub authentication errors, API rate limits, malformed HTML content, network issues.  
  - Notes: Content is escaped to avoid JSON parse errors.  
  - Version: HTTP Request node v4.1 supporting JSON body and OAuth.

---

#### 2.3 Telegram Notification

**Overview:**  
This block sends a Telegram message to a specified chat notifying of the new email, including a clickable inline button that links to the temporary HTML page hosted on a custom domain using the gist ID.

**Nodes Involved:**  
- Telegram (Send)  
- Sticky Note (Documentation)

**Node Details:**  

- **Telegram (Send)**  
  - Type: Telegram node (Send Message)  
  - Role: Sends a formatted HTML message to a specified Telegram chat with an inline keyboard button linking to the hosted email HTML page.  
  - Configuration:  
    - Chat ID: User-provided Telegram chat ID.  
    - Text: HTML formatted message notifying about the new email, dynamically including the sender's email address.  
    - Parse Mode: HTML (to allow `<b>`, `<code>`, etc.)  
    - Disable Web Page Preview: true (prevents Telegram from generating link previews)  
    - Inline Keyboard: One button labeled with the filename (`email.html`), linking to `http://emails.nskha.com/?iloven8n=nskha&id={gist_id}` (custom domain with gist ID as a query parameter).  
  - Input: Output of Github Gist node for the gist ID and filename.  
  - Output: Passes data to the Wait node for cleanup scheduling.  
  - Failure Types: Telegram API errors due to invalid chat ID, message formatting issues, network timeouts, special character restrictions.  
  - Version: Telegram node v1.1 supporting inline keyboards and HTML parse mode.  

- **Sticky Note**  
  - Type: Documentation node  
  - Content: Instructions on credential setup, optional project hosting, branding images, and links.  
  - Context: Provides users with setup tips and references.  

---

#### 2.4 Timed Cleanup

**Overview:**  
This block waits for a specified time (default 3 hours), then deletes the GitHub Gist and removes the Telegram notification message to keep email previews temporary and secure.

**Nodes Involved:**  
- Wait  
- Github Gist (Delete)  
- Telegram (Delete Message)

**Node Details:**  

- **Wait**  
  - Type: Wait node  
  - Role: Delays the workflow for 3 hours before triggering cleanup.  
  - Configuration:  
    - Amount: 3 (hours)  
  - Input: Output from Telegram notification node.  
  - Output: Triggers the GitHub Gist deletion node.  
  - Failure Types: Workflow timeouts with very long waits, manual cancellations.  
  - Notes: Duration can be adjusted as needed.  

- **Github Gist (Delete)**  
  - Type: HTTP Request (DELETE to GitHub Gist API)  
  - Role: Deletes the previously created private gist using its ID to remove the hosted HTML page.  
  - Configuration:  
    - URL: `https://api.github.com/gists/{gist_id}` dynamically extracted from the original gist creation node.  
    - Method: DELETE  
    - Authentication: Predefined GitHub API credential (token with gist deletion permission)  
    - Headers: Accept header set to `application/vnd.github+json`  
  - Input: Wait node output, uses original gist ID.  
  - Output: Passes result to Telegram message deletion node.  
  - Failure Types: Authorization errors, gist not found (already deleted), network failures.  

- **Telegram (Delete Message)**  
  - Type: Telegram node (Delete Message)  
  - Role: Deletes the original Telegram notification message to clean up the chat.  
  - Configuration:  
    - Chat ID: Same as used for sending the message.  
    - Message ID: Extracted from the first Telegram node’s output (`message_id`).  
    - Operation: Delete message  
  - Input: Output from GitHub Gist deletion node.  
  - Output: End of workflow.  
  - Failure Types: Message already deleted, invalid chat ID, API errors.  

---

### 3. Summary Table

| Node Name          | Node Type                 | Functional Role                          | Input Node(s)          | Output Node(s)          | Sticky Note                                                                                                                                        |
|--------------------|---------------------------|----------------------------------------|-----------------------|------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------|
| Email Trigger (IMAP)| Email Trigger (IMAP)      | Detect new unread emails                | None                  | Github Gist            |                                                                                                                                                |
| Github Gist        | HTTP Request              | Create private GitHub Gist for email HTML | Email Trigger (IMAP)   | Telegram               | Save HTML content                                                                                                                                  |
| Telegram           | Telegram                  | Send Telegram notification with link  | Github Gist           | Wait                   |                                                                                                                                                |
| Wait               | Wait                      | Wait 3 hours before cleanup            | Telegram               | Github Gist ‌           | Delete within 3h                                                                                                                                   |
| Github Gist ‌       | HTTP Request              | Delete the GitHub Gist                 | Wait                   | Telegram ‌             | Remove HTML content                                                                                                                                |
| Telegram ‌          | Telegram                  | Delete Telegram notification message  | Github Gist ‌          | None                   |                                                                                                                                                |
| Sticky Note        | Sticky Note               | Setup instructions and branding        | None                  | None                   | ## Simple Conversion of Emails into HTML Webpages To-do: * Configure your GitHub credentials through `Predefined Credential Type` => `GitHub API`. * Add your Telegram credentials by providing your `Chat ID`. * [**Optional**] You can host this [small project](https://github.com/Automations-Project/Emails/tree/main) on your own domain using GitHub Pages. ![image](https://cdn.statically.io/gh/Automations-Project/Emails/main/iloven8n.min.svg) ![image](https://cdn.statically.io/gh/Automations-Project/Emails/main/iloven8n%E2%80%8C.min.svg) |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Email Trigger (IMAP) node:**  
   - Node Type: Email Trigger (IMAP)  
   - Set email credentials (IMAP server, port, username, password).  
   - Configure to only trigger on unread emails: add custom config JSON `["UNSEEN"]`.  
   - Set format to “resolved” to get full parsed email content including HTML.  
   - Set forceReconnect interval to 60 seconds.  
   - Position: starting point of the workflow.

2. **Add HTTP Request node to create GitHub Gist:**  
   - Node Type: HTTP Request  
   - Name it “Github Gist”  
   - Method: POST  
   - URL: `https://api.github.com/gists`  
   - Authentication: Use a predefined GitHub API credential with permissions to create gists.  
   - Headers: Add header `Accept: application/vnd.github+json`.  
   - Body Type: JSON  
   - Body Content:  
     ```json
     {
       "description": "{{ $json.date }} - from {{ JSON.stringify($json.from.value[0].address).slice(1, -1) }} - to {{ JSON.stringify($json.to.value[0].address).slice(1, -1) }}",
       "public": false,
       "files": {
         "email.html": {
           "content": "{{ JSON.stringify($json.html).slice(1, -1) }}"
         }
       }
     }
     ```  
   - Connect Email Trigger (IMAP) node output to this node.

3. **Add Telegram node to send notification:**  
   - Node Type: Telegram  
   - Chat ID: Your Telegram chat ID  
   - Text:  
     ```
     📧 <b>You've got mail!</b>

     A new email has arrived from this address: <code>{{ $node["Email Trigger (IMAP)"].json["from"]["value"][0]["address"] }}</code>

     🌐 A secret HTML page has been created for it, where you can preview the message by following the link below 👇
     ```  
   - Parse Mode: HTML  
   - Disable Web Page Preview: true  
   - Reply Markup: Inline Keyboard with one button:  
     - Text: `={{ $('Github Gist').item.json.files["email.html"].filename }}`  
     - URL: `={{'http://emails.nskha.com/?iloven8n=nskha&id='+ $('Github Gist').item.json.id}}`  
   - Connect Github Gist node output to this Telegram node.

4. **Add Wait node:**  
   - Node Type: Wait  
   - Amount: 3 hours (or your preferred delay)  
   - Connect Telegram node output to Wait node.

5. **Add HTTP Request node to delete GitHub Gist:**  
   - Node Type: HTTP Request  
   - Name: “Github Gist ‌” (Delete)  
   - Method: DELETE  
   - URL: `https://api.github.com/gists/{{ $item("0").$node["Github Gist"].json["id"] }}`  
   - Authentication: Same GitHub API credential as creation node.  
   - Headers: `Accept: application/vnd.github+json`  
   - Connect Wait node output to this node.

6. **Add Telegram node to delete the notification message:**  
   - Node Type: Telegram  
   - Operation: Delete Message  
   - Chat ID: Same as send message node  
   - Message ID: `={{ $('Telegram').item.json.result.message_id }}` (extract from initial send message response)  
   - Connect Github Gist deletion node output to this node.

7. **Add Sticky Note node (optional):**  
   - Add instructions on GitHub and Telegram credential setup, warnings about privacy, and helpful links.  
   - Position it visibly near the start nodes for user reference.

8. **Connect all nodes in the order specified:**  
   Email Trigger (IMAP) → Github Gist (create) → Telegram (send) → Wait → Github Gist (delete) → Telegram (delete message)

9. **Credential Setup:**  
   - Configure IMAP credentials for your mailbox in Email Trigger node.  
   - Configure Telegram credentials (bot token) and set Chat ID in Telegram nodes.  
   - Configure GitHub API credentials with `gist` create/delete permissions for HTTP Request nodes.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                         | Context or Link                                                                                      |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| Optional: Host the project on your own domain using GitHub Pages for custom HTML preview URLs.                                                                                                                                     | https://github.com/Automations-Project/Emails/tree/main                                            |
| Ensure security by using private GitHub Gists and deleting them after a set delay to avoid data exposure.                                                                                                                          | Workflow design principle                                                                           |
| Test the workflow with non-sensitive emails first before connecting critical email accounts.                                                                                                                                       | Best practice                                                                                      |
| For Telegram API special characters handling limitations, rendering email content as HTML in a webpage bypasses Telegram's text restrictions and preserves formatting.                                                               | Workflow rationale                                                                                 |
| GitHub tokens must have `gist` scope enabled for creation and deletion of Gists.                                                                                                                                                    | GitHub Documentation: https://docs.github.com/en/rest/gists/gists#create-a-gist                     |
| Telegram API documentation for message sending and deletion: https://core.telegram.org/bots/api                                                                                                                                     | Telegram Bot API                                                                                   |
| IMAP connection stability can be improved by setting `forceReconnect` option in Email Trigger node.                                                                                                                                 | n8n Email Trigger (IMAP) node docs                                                                 |

---

This structured reference document provides detailed insights into each component of the workflow, enabling users or automation agents to understand, reproduce, and maintain the workflow efficiently.