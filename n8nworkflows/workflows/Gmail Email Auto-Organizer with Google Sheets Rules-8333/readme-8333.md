Gmail Email Auto-Organizer with Google Sheets Rules

https://n8nworkflows.xyz/workflows/gmail-email-auto-organizer-with-google-sheets-rules-8333


# Gmail Email Auto-Organizer with Google Sheets Rules

### 1. Workflow Overview

This workflow automatically organizes incoming Gmail messages based on rules defined in a Google Sheets document. It targets users who want to declutter their Gmail inbox by automatically categorizing, labeling, marking as read, or removing promotional and other categorized emails according to dynamic, user-maintained rules.

The workflow logic is organized into these main blocks:

- **1.1 Input Reception**: Triggered periodically, fetches all new Gmail messages and the current Gmail labels.
- **1.2 Sender Email Parsing**: Extracts and normalizes sender email addresses from each message.
- **1.3 Marketing/Automated Email Detection**: Classifies emails as promotional or automated based on sender patterns.
- **1.4 Rule Retrieval and Application**: Retrieves user-defined email handling rules from Google Sheets and applies them to each message.
- **1.5 Label Mapping and Validation**: Maps label names from rules to Gmail label IDs and checks label existence.
- **1.6 Email Labeling and Inbox Management**: Applies labels, removes emails from the inbox, and marks them as read accordingly.
- **1.7 Completion Notification**: Sends a Slack notification when the organization process completes.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:** Periodically triggers the workflow, fetches all inbox messages and Gmail labels to start processing.
- **Nodes Involved:** Schedule Trigger, Get many messages, Gmail Labels, Merge

1. **Schedule Trigger**
   - Type: Schedule Trigger
   - Role: Initiates the workflow every minute.
   - Configuration: Interval set to trigger every minute.
   - Inputs: None
   - Outputs: Triggers "Get many messages" and "Gmail Labels" nodes simultaneously.
   - Potential Failures: Misconfiguration of schedule, n8n instance downtime.

2. **Get many messages**
   - Type: Gmail node (OAuth2)
   - Role: Fetches all messages labeled as INBOX.
   - Configuration: Operation 'getAll', filter by label 'INBOX'.
   - Credentials: OAuth2 Gmail account.
   - Inputs: Trigger from Schedule Trigger.
   - Outputs: List of inbox messages.
   - Potential Failures: OAuth2 token expiration, Gmail API rate limits, connection issues.

3. **Gmail Labels**
   - Type: Gmail node (OAuth2)
   - Role: Retrieves all Gmail labels for mapping.
   - Configuration: Resource 'label', return all labels.
   - Credentials: OAuth2 Gmail account.
   - Inputs: Trigger from Schedule Trigger.
   - Outputs: List of Gmail labels.
   - Potential Failures: OAuth2 issues, API limits.

4. **Merge**
   - Type: Merge node
   - Role: Merges Gmail labels with processed messages later in the workflow.
   - Inputs: Linked downstream after label mapping block.
   - Outputs: Combined dataset for further processing.

---

#### 2.2 Sender Email Parsing

- **Overview:** Extracts the sender's email address from the raw "From" field and normalizes it for matching.
- **Nodes Involved:** Parse Sender Email

1. **Parse Sender Email**
   - Type: Code node (JavaScript)
   - Role: Extracts clean sender email addresses from raw "From" fields.
   - Configuration: Custom JS extracts the email part inside angle brackets or uses raw if none.
   - Key Expression: Uses regex `<([^>]+)>` to extract email.
   - Inputs: Messages from "Get many messages".
   - Outputs: Messages enriched with `_fromEmail` property.
   - Potential Failures: Unexpected email formats, missing "From" fields.

---

#### 2.3 Marketing/Automated Email Detection

- **Overview:** Detects if an email is marketing or automated by matching sender against a predefined list of patterns.
- **Nodes Involved:** Is marketing/automated email?, Promotional, Sheet Rules, Merge1

1. **Is marketing/automated email?**
   - Type: If node
   - Role: Checks if sender email matches patterns that indicate marketing/automated emails.
   - Configuration: Uses JavaScript expression checking sender email against known promotion senders (e.g., newsletters, LinkedIn, Slack).
   - Inputs: Messages from "Loop Over Items".
   - Outputs: True branch to "Promotional", False branch to "Sheet Rules".
   - Potential Failures: Expression errors, false positives/negatives if patterns outdated.

2. **Promotional**
   - Type: Gmail node
   - Role: Adds the "CATEGORY_PROMOTIONS" label to promotional emails.
   - Configuration: Label ID "CATEGORY_PROMOTIONS" added to message ID.
   - Inputs: True branch from "Is marketing/automated email?".
   - Outputs: Connects to "Remove From Inbox".
   - Potential Failures: Label might not exist or API errors.

3. **Sheet Rules**
   - Type: Google Sheets node (OAuth2)
   - Role: Reads user-defined email handling rules from the first sheet of a specific Google Sheet.
   - Configuration: Reads all rows from sheet named "Sheet1" in given document ID.
   - Credentials: Google Sheets OAuth2.
   - Inputs: False branch from "Is marketing/automated email?".
   - Outputs: Sends rules data to "Merge1".
   - Potential Failures: OAuth2 issues, sheet access permissions, empty or malformed sheets.

4. **Merge1**
   - Type: Merge node (chooseBranch mode)
   - Role: Combines the email items with rules for further processing.
   - Inputs: Receives data from "Sheet Rules" and "Is marketing/automated email?".
   - Outputs: To "Apply Sheet Rules".
   - Potential Failures: Branch mismatch or empty input.

---

#### 2.4 Rule Application and Label Mapping

- **Overview:** Applies Google Sheets rules to each email, determines appropriate action, and maps label names to Gmail label IDs.
- **Nodes Involved:** Apply Sheet Rules, Merge, Map Label Name, is Label Exist?

1. **Apply Sheet Rules**
   - Type: Code node (JavaScript)
   - Role: Matches each sender email against Google Sheets rules and assigns an action: DELETE, PROMO, LABEL, or 'extra' (default).
   - Configuration: Parses rows from "Sheet Rules" node, filters by pattern matching, sets `switchLable` for downstream processing.
   - Inputs: Email items with rules from "Merge1".
   - Outputs: Items with rule metadata attached.
   - Potential Failures: Rule format errors, missing fields causing null hits.

2. **Merge**
   - Type: Merge node
   - Role: Combines output from "Apply Sheet Rules" with Gmail labels from "Gmail Labels" for label ID mapping.
   - Inputs: From "Apply Sheet Rules" and "Gmail Labels".
   - Outputs: To "Map Label Name".
   - Potential Failures: Synchronization mismatches.

3. **Map Label Name**
   - Type: Code node (JavaScript)
   - Role: Maps human-readable label names from rules to Gmail label IDs using the label list.
   - Configuration: Builds a name-to-ID map, assigns label IDs if missing, skipping PROMO, DELETE, and 'extra'.
   - Inputs: Merged data from previous step.
   - Outputs: To "is Label Exist?" node.
   - Potential Failures: Missing labels, case sensitivity, empty label lists.

4. **is Label Exist?**
   - Type: If node
   - Role: Checks if the label ID exists for each item.
   - Configuration: Checks if `labelId` is not empty.
   - Inputs: From "Map Label Name".
   - Outputs: True branch triggers "Add Label", False branch loops back to "Loop Over Items".
   - Potential Failures: Logic errors leading to infinite loops.

---

#### 2.5 Email Labeling and Inbox Management

- **Overview:** Applies labels, removes emails from the inbox, and marks them as read based on the classification and rules.
- **Nodes Involved:** Add Label, Remove From Inbox, Mark as read, Loop Over Items

1. **Add Label**
   - Type: Gmail node
   - Role: Adds the specified label to a message.
   - Configuration: Adds label(s) by ID extracted from previous steps.
   - Inputs: True branch from "is Label Exist?".
   - Outputs: To "Remove From Inbox".
   - Potential Failures: Invalid label IDs, API failures.

2. **Remove From Inbox**
   - Type: Gmail node
   - Role: Removes the INBOX label from a message, effectively archiving it.
   - Configuration: Removes "INBOX" label from messages.
   - Inputs: From "Add Label" and "Promotional" nodes.
   - Outputs: To "Mark as read".
   - Potential Failures: API limits, label removal errors.

3. **Mark as read**
   - Type: Gmail node
   - Role: Marks the email as read.
   - Configuration: Marks message as read by message ID.
   - Inputs: From "Remove From Inbox".
   - Outputs: Loops back to "Loop Over Items" to process next batch.
   - Potential Failures: API issues.

4. **Loop Over Items**
   - Type: SplitInBatches node
   - Role: Processes emails in batches to handle large volumes safely.
   - Configuration: Default batch size.
   - Inputs: From "Parse Sender Email" and loopbacks.
   - Outputs: To "Is marketing/automated email?" and "Completed Notification".
   - Potential Failures: Batch size too large causing timeouts.

---

#### 2.6 Completion Notification

- **Overview:** Sends a Slack message notifying that the email organization process has completed.
- **Nodes Involved:** Completed Notification

1. **Completed Notification**
   - Type: Slack node
   - Role: Posts a completion message to a specific Slack channel.
   - Configuration: Message text "✅ Email Organization completed", channel ID provided.
   - Credentials: Slack API OAuth2.
   - Inputs: From "Loop Over Items" (executed once per run).
   - Outputs: None (terminal node).
   - Potential Failures: Slack API token expiration, channel ID invalid.

---

### 3. Summary Table

| Node Name                 | Node Type              | Functional Role                         | Input Node(s)                  | Output Node(s)              | Sticky Note                                                                                          |
|---------------------------|------------------------|---------------------------------------|-------------------------------|-----------------------------|----------------------------------------------------------------------------------------------------|
| Schedule Trigger          | Schedule Trigger       | Triggers workflow every minute        | None                          | Get many messages, Gmail Labels |                                                                                                    |
| Get many messages         | Gmail                  | Fetches inbox emails                   | Schedule Trigger              | Parse Sender Email           |                                                                                                    |
| Parse Sender Email        | Code                   | Extracts normalized sender emails     | Get many messages             | Loop Over Items              |                                                                                                    |
| Loop Over Items           | SplitInBatches         | Processes emails in batches            | Parse Sender Email            | Is marketing/automated email?, Completed Notification |                                                                                                    |
| Is marketing/automated email? | If                    | Detects marketing/promotional senders | Loop Over Items               | Promotional (true), Sheet Rules (false), Merge1 (false) |                                                                                                    |
| Promotional               | Gmail                  | Labels promotional emails              | Is marketing/automated email? | Remove From Inbox            |                                                                                                    |
| Sheet Rules               | Google Sheets          | Reads user-defined email rules        | Is marketing/automated email? | Merge1                      |                                                                                                    |
| Merge1                   | Merge                  | Combines emails and rules              | Sheet Rules, Is marketing/automated email? | Apply Sheet Rules           |                                                                                                    |
| Apply Sheet Rules         | Code                   | Applies Google Sheets rules to emails | Merge1                       | Merge                       |                                                                                                    |
| Gmail Labels              | Gmail                  | Retrieves Gmail labels                 | Schedule Trigger              | Merge                       |                                                                                                    |
| Merge                    | Merge                  | Combines labeled emails with Gmail labels | Apply Sheet Rules, Gmail Labels | Map Label Name              |                                                                                                    |
| Map Label Name            | Code                   | Maps label names to Gmail label IDs   | Merge                        | is Label Exist?              |                                                                                                    |
| is Label Exist?           | If                     | Checks if label ID exists              | Map Label Name               | Add Label (true), Loop Over Items (false) |                                                                                                    |
| Add Label                 | Gmail                  | Adds label to email                    | is Label Exist?               | Remove From Inbox            |                                                                                                    |
| Remove From Inbox         | Gmail                  | Removes INBOX label (archives email)  | Add Label, Promotional        | Mark as read                |                                                                                                    |
| Mark as read              | Gmail                  | Marks email as read                    | Remove From Inbox             | Loop Over Items              |                                                                                                    |
| Completed Notification    | Slack                  | Sends completion message               | Loop Over Items              | None                        |                                                                                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node:**
   - Set interval to every 1 minute.

2. **Add Gmail node "Get many messages":**
   - Operation: getAll
   - Filter by label: INBOX
   - Authenticate with Gmail OAuth2 credentials.

3. **Add Gmail node "Gmail Labels":**
   - Resource: label
   - Return all labels
   - Use same Gmail OAuth2 credentials.

4. **Connect "Schedule Trigger" output to both "Get many messages" and "Gmail Labels".**

5. **Add Code node "Parse Sender Email":**
   - JavaScript to extract email from "From" field:
     ```js
     return items.map(it => {
       const raw = String(it.json.From || "");
       const email = (raw.toLowerCase().match(/<([^>]+)>/)?.[1] || raw.toLowerCase()).trim();
       it.json._fromEmail = email;
       return it;
     });
     ```
   - Connect from "Get many messages".

6. **Add SplitInBatches node "Loop Over Items":**
   - Default options.
   - Connect from "Parse Sender Email".

7. **Add If node "Is marketing/automated email?":**
   - Use JavaScript expression to check if sender email contains known promotional patterns:
     ```js
     (() => {
       const fromRaw = ($json["From"] || "").toString().toLowerCase();
       const fromEmail = (fromRaw.match(/<([^>]+)>/)?.[1] || fromRaw).trim();
       const patterns = [
         "ivan@mail.notion.so", "notifications@discord.com",
         "newsletter@", "mailer@", "bounce@",
         "@linkedin.com", "@e.linkedin.com", "@facebookmail.com",
         "@mail.instagram.com", "@slack.com", "@mailchimp.com",
         "info@e.atlassian.com",
         "x.com",
         "navicosoft.com"
       ];
       return patterns.some(p => fromEmail.includes(p));
     })()
     ```
   - Connect from "Loop Over Items".

8. **Add Gmail node "Promotional":**
   - Operation: addLabels
   - Label: CATEGORY_PROMOTIONS
   - Connect from True branch of "Is marketing/automated email?"
   - Use Gmail OAuth2 credentials.

9. **Add Gmail node "Remove From Inbox":**
   - Operation: removeLabels
   - Label: INBOX
   - Connect from "Promotional" and later from "Add Label".
   - Use Gmail OAuth2 credentials.

10. **Add Gmail node "Mark as read":**
    - Operation: markAsRead
    - Connect from "Remove From Inbox".
    - Use Gmail OAuth2 credentials.

11. **Connect "Mark as read" output back to "Loop Over Items" to continue batches.**

12. **Add Google Sheets node "Sheet Rules":**
    - Read rows from specific Google Sheet and sheet name "Sheet1".
    - Authenticate with Google Sheets OAuth2 credentials.
    - Connect from False branch of "Is marketing/automated email?".

13. **Add Merge node "Merge1" in chooseBranch mode:**
    - Connect inputs from "Sheet Rules" and False branch of "Is marketing/automated email?".

14. **Add Code node "Apply Sheet Rules":**
    - JavaScript to apply rules:
      - Parse sheet rows to rules.
      - Match sender email to pattern.
      - Set action: DELETE, PROMO, LABEL, or extra.
      - Assign labelId and switchLabel accordingly.
    - Connect from "Merge1".

15. **Add Merge node "Merge":**
    - Connect inputs from "Apply Sheet Rules" and "Gmail Labels".

16. **Add Code node "Map Label Name":**
    - Build map of labelName → labelId.
    - Assign labelId if missing and labelName valid.
    - Connect from "Merge".

17. **Add If node "is Label Exist?":**
    - Condition: labelId is not empty.
    - Connect from "Map Label Name".

18. **Add Gmail node "Add Label":**
    - Operation: addLabels
    - Label IDs from item.
    - Connect from True branch of "is Label Exist?".

19. **Connect "Add Label" output to "Remove From Inbox" node.**

20. **Connect False branch of "is Label Exist?" back to "Loop Over Items" to continue.**

21. **Add Slack node "Completed Notification":**
    - Text: "✅ Email Organization completed"
    - Channel: Provide Slack channel ID.
    - Authenticate with Slack OAuth2 credentials.
    - Connect from "Loop Over Items" execute once node.

---

### 5. General Notes & Resources

| Note Content                                                                                     | Context or Link                                                                               |
|-------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------|
| Gmail API OAuth2 credentials must have scopes to read, modify, and label messages.               | Gmail OAuth2 with scope: https://www.googleapis.com/auth/gmail.modify                       |
| Google Sheets OAuth2 credentials require access to the specific spreadsheet for rules reading.   | Google Sheets API with proper sharing permissions                                           |
| Slack OAuth2 token must have permission to post messages to the specified channel.               | Slack API scopes: chat:write                                                                 |
| Promotional email detection patterns are hardcoded and can be customized in the "Is marketing/automated email?" node | Adjust patterns to fit user-specific promotional senders                                    |
| Batch processing via SplitInBatches ensures safe API usage within Gmail rate limits.             | Default batch size, can be adjusted if hitting limits                                       |
| The workflow assumes labels exist in Gmail or are pre-created as per Google Sheets rules.       | Missing labels might prevent labeling or cause errors                                       |
| Slack notification node uses executeOnce to send a single message after batch processing finishes | Useful for monitoring workflow health                                                      |

---

This detailed documentation allows expert users and automation agents to fully understand, reproduce, and maintain the Gmail Email Auto-Organizer workflow integrating Google Sheets for dynamic rule management.