Route email actions from Notion with Gmail, Slack, and Jira

https://n8nworkflows.xyz/workflows/route-email-actions-from-notion-with-gmail--slack--and-jira-14120


# Route email actions from Notion with Gmail, Slack, and Jira

## 1. Workflow Overview

This workflow, titled **Email Router**, polls a Notion database that acts as an **Email Intelligence queue** and executes downstream actions based on each record’s **Status**. It is designed for product, operations, or support teams that triage incoming emails in Notion and then want automation to carry out the selected action.

Primary use cases:
- Send a prepared reply through Gmail
- Forward an email to a delegate and notify them in Slack
- Route an email-derived signal into another Notion database for downstream processing
- Archive an email logically and mark the originating Notion record as processed

### 1.1 Intake from Notion
The workflow starts by reading pages from a Notion database and extracting normalized email-routing fields from the page properties.

### 1.2 Action Selection by Status
A central switch checks the Notion page’s `Status` property and branches into one of four action paths:
- Responded
- Delegated
- Routed
- Archived

### 1.3 Responded Path
If the item is marked as responded, the workflow sends the prepared draft via Gmail and then updates the original Notion entry to `Processed`.

### 1.4 Delegated Path
If the item is delegated, the workflow forwards context by email, sends a Slack DM-style notification, and then updates the Notion record to `Processed`.

### 1.5 Routed Path
If the item is routed, the workflow determines the route destination using tags/category heuristics, prepares a normalized signal payload, creates a page in a destination Notion database, and then marks the original item as `Processed`.

### 1.6 Archived Path
If the item is archived, the workflow prepares Gmail label/archive metadata, then updates the Notion record to `Processed`. In its current form, this branch prepares archive instructions but does **not** actually call Gmail to modify labels.

---

## 2. Block-by-Block Analysis

## 2.1 Intake from Notion

**Overview:**  
This block retrieves candidate email-intelligence pages from Notion and transforms the raw page structure into a flatter JSON object that is easier to route and process in later nodes.

**Nodes Involved:**  
- Poll Email Intelligence
- Extract Email Data

### Node: Poll Email Intelligence
- **Type and technical role:**  
  `n8n-nodes-base.notion` — reads pages from a Notion database.
- **Configuration choices:**  
  - Resource: `databasePage`
  - Operation: `getAll`
  - Database ID: `31e06bab-3ebe-81d6-8c12-f607c8ff3b3f`
  - Limit: `10`
  - Filter type: `formula`
- **Key expressions or variables used:**  
  No custom expression is used in the visible parameters except the database ID reference wrapper.
- **Input and output connections:**  
  - Input: none; this is the workflow’s effective starting node
  - Output: `Extract Email Data`
- **Version-specific requirements:**  
  Uses Notion node `typeVersion 2.2`.
- **Edge cases or potential failure types:**  
  - Notion credential/authentication failure
  - Database ID invalid or inaccessible
  - Formula filter expected but not fully configured in visible parameters; behavior may depend on n8n UI defaults or imported settings
  - Pagination/limit may exclude items if more than 10 are pending
- **Sub-workflow reference:**  
  None

### Node: Extract Email Data
- **Type and technical role:**  
  `n8n-nodes-base.code` — maps Notion page properties into a normalized email-routing object.
- **Configuration choices:**  
  Custom JavaScript extracts these fields:
  - `pageId`
  - `notionUrl`
  - `subject`
  - `from`
  - `category`
  - `urgency`
  - `status`
  - `summary`
  - `keyAsk`
  - `draftResponse`
  - `customer`
  - `revenueImpact`
  - `suggestedAction`
  - `delegateTo`
  - `emailId`
  - `threadId`
  - `tags`
- **Key expressions or variables used:**  
  - `$input.item.json`
  - Optional chaining on Notion properties, for example:  
    `page.properties?.['Subject']?.title?.[0]?.plain_text || ''`
- **Input and output connections:**  
  - Input: `Poll Email Intelligence`
  - Output: `Switch by Status`
- **Version-specific requirements:**  
  Uses Code node `typeVersion 2`.
- **Edge cases or potential failure types:**  
  - Missing or renamed Notion properties produce empty strings/default values
  - Multi-value rich text/title/select fields are truncated to first value only
  - If downstream nodes require `emailId`, `threadId`, or `draftResponse`, empty defaults may lead to execution issues later
- **Sub-workflow reference:**  
  None

---

## 2.2 Action Selection by Status

**Overview:**  
This block is the workflow’s main decision point. It routes each normalized email record into one of four branches based on its `status`.

**Nodes Involved:**  
- Switch by Status

### Node: Switch by Status
- **Type and technical role:**  
  `n8n-nodes-base.switch` — conditional router with four outputs.
- **Configuration choices:**  
  Compares `{{$json.status}}` to:
  1. `Responded`
  2. `Delegated`
  3. `Routed`
  4. `Archived`
  
  Fallback output is set to `none`, so unmatched items stop here.
- **Key expressions or variables used:**  
  - `={{ $json.status }}`
- **Input and output connections:**  
  - Input: `Extract Email Data`
  - Outputs:
    - Output 0 → `Gmail Send Reply`
    - Output 1 → `Gmail Forward`
    - Output 2 → `Switch by Route Destination`
    - Output 3 → `Prepare Archive Labels`
- **Version-specific requirements:**  
  Uses Switch node `typeVersion 3.2`.
- **Edge cases or potential failure types:**  
  - Status values must match exactly, including capitalization
  - Any non-matching status is silently dropped because fallback is `none`
  - If Notion uses `Processed` or another intermediate status directly, the record will not continue
- **Sub-workflow reference:**  
  None

---

## 2.3 Responded Path

**Overview:**  
This branch sends the draft response through Gmail and then updates the source Notion page to indicate processing is complete.

**Nodes Involved:**  
- Gmail Send Reply
- Update Status - Responded

### Node: Gmail Send Reply
- **Type and technical role:**  
  `n8n-nodes-base.gmail` — sends an email reply message.
- **Configuration choices:**  
  - Subject: `Re: {{ $json.subject }}`
  - Message body: `{{ $json.draftResponse }}`
  - No additional options are configured
- **Key expressions or variables used:**  
  - `={{ $json.draftResponse }}`
  - `=Re: {{ $json.subject }}`
- **Input and output connections:**  
  - Input: `Switch by Status` (Responded branch)
  - Output: `Update Status - Responded`
- **Version-specific requirements:**  
  Uses Gmail node `typeVersion 2.1`.
- **Edge cases or potential failure types:**  
  - Gmail OAuth2 credentials missing or expired
  - Empty `draftResponse` sends a blank or near-blank email
  - This node does not visibly use `threadId`, `emailId`, or recipient fields, so it may not behave as a true threaded reply without additional configuration
  - Subject-only reply logic can create a new message instead of replying in-thread depending on Gmail node settings
- **Sub-workflow reference:**  
  None

### Node: Update Status - Responded
- **Type and technical role:**  
  `n8n-nodes-base.notion` — updates the originating Notion page after a successful send.
- **Configuration choices:**  
  - Resource: `databasePage`
  - Operation: `update`
  - Page ID: from `Extract Email Data`
  - Sets:
    - `Status` = `Processed`
    - `Processed At` = current timestamp
    - `Action Taken` rich text field present but no explicit value shown
- **Key expressions or variables used:**  
  - `={{ $('Extract Email Data').item.json.pageId }}`
  - `={{ $now.toISO() }}`
- **Input and output connections:**  
  - Input: `Gmail Send Reply`
  - Output: none
- **Version-specific requirements:**  
  Uses Notion node `typeVersion 2.2`.
- **Edge cases or potential failure types:**  
  - Fails if page no longer exists or permissions changed
  - If `Action Taken` is required in Notion schema, blank update behavior should be validated
  - Partial success pattern: email may send successfully but Notion update may fail, causing reprocessing risk on next poll
- **Sub-workflow reference:**  
  None

---

## 2.4 Delegated Path

**Overview:**  
This branch sends a forwarding email with summary/context, then sends a Slack notification to the delegate, and finally marks the original item as processed in Notion.

**Nodes Involved:**  
- Gmail Forward
- Slack DM Delegate
- Update Status - Delegated

### Node: Gmail Forward
- **Type and technical role:**  
  `n8n-nodes-base.gmail` — sends a contextual forwarding message.
- **Configuration choices:**  
  - Subject: `FWD: {{ $json.subject }}`
  - Message body is a formatted plain-text/Markdown-like template including:
    - Summary
    - Key Ask
    - Urgency
    - Customer if available
    - Revenue Impact if available
- **Key expressions or variables used:**  
  - `=FWD: {{ $json.subject }}`
  - Body uses:
    - `{{ $json.summary }}`
    - `{{ $json.keyAsk }}`
    - `{{ $json.urgency }}`
    - conditional inserts for `customer` and `revenueImpact`
- **Input and output connections:**  
  - Input: `Switch by Status` (Delegated branch)
  - Output: `Slack DM Delegate`
- **Version-specific requirements:**  
  Uses Gmail node `typeVersion 2.1`.
- **Edge cases or potential failure types:**  
  - No explicit recipient field is shown; unless configured elsewhere in credentials/defaults, this may fail or be incomplete
  - Markdown-style formatting may not render as intended in email clients
  - Empty `summary` or `keyAsk` reduces usefulness of the forwarded message
  - If the intent is actual message forwarding from original Gmail email/thread, this configuration is composing a new email, not necessarily forwarding the existing original
- **Sub-workflow reference:**  
  None

### Node: Slack DM Delegate
- **Type and technical role:**  
  `n8n-nodes-base.slack` — posts a Slack message containing delegation details.
- **Configuration choices:**  
  The message text includes:
  - Subject
  - From
  - Urgency
  - Summary
  - Key Ask
  - A note that the full email was forwarded
- **Key expressions or variables used:**  
  References the earlier extracted item explicitly:
  - `{{ $('Extract Email Data').item.json.subject }}`
  - `{{ $('Extract Email Data').item.json.from }}`
  - `{{ $('Extract Email Data').item.json.urgency }}`
  - `{{ $('Extract Email Data').item.json.summary }}`
  - `{{ $('Extract Email Data').item.json.keyAsk }}`
- **Input and output connections:**  
  - Input: `Gmail Forward`
  - Output: `Update Status - Delegated`
- **Version-specific requirements:**  
  Uses Slack node `typeVersion 2.2`.
- **Edge cases or potential failure types:**  
  - No explicit Slack recipient/channel/user ID is visible
  - “DM delegate” behavior depends on node operation defaults or hidden config; if not properly configured, the message may fail or post to the wrong destination
  - Slack auth, bot scopes, or workspace restrictions may block sending
- **Sub-workflow reference:**  
  None

### Node: Update Status - Delegated
- **Type and technical role:**  
  `n8n-nodes-base.notion` — marks the source Notion page as processed after delegation.
- **Configuration choices:**  
  - Page ID from `Extract Email Data`
  - Sets:
    - `Status` = `Processed`
    - `Processed At` = current timestamp
    - `Action Taken` rich text field present but blank in visible config
- **Key expressions or variables used:**  
  - `={{ $('Extract Email Data').item.json.pageId }}`
  - `={{ $now.toISO() }}`
- **Input and output connections:**  
  - Input: `Slack DM Delegate`
  - Output: none
- **Version-specific requirements:**  
  Uses Notion node `typeVersion 2.2`.
- **Edge cases or potential failure types:**  
  Same main risks as other Notion update nodes:
  - page permission/access changes
  - post-action Notion update failure causing duplicate processing risk
- **Sub-workflow reference:**  
  None

---

## 2.5 Routed Path

**Overview:**  
This branch is intended to route email-derived signals to downstream destinations based on tags or category. In the current implementation, all route conditions converge into the same “prepare and create in Signal Stream” path.

**Nodes Involved:**  
- Switch by Route Destination
- Prepare Route Data
- Create in Signal Stream
- Update Status - Routed

### Node: Switch by Route Destination
- **Type and technical role:**  
  `n8n-nodes-base.switch` — evaluates route heuristics for routed items.
- **Configuration choices:**  
  Four route rules:
  1. `tags` contains `RICE`
  2. `category` contains `Customer`
  3. `tags` contains `Sprint`
  4. `category` equals `Competitor Intel`
  
  All four outputs currently go to the same next node.
- **Key expressions or variables used:**  
  - `={{ $json.tags }}`
  - `={{ $json.category }}`
- **Input and output connections:**  
  - Input: `Switch by Status` (Routed branch)
  - Outputs 0–3 → `Prepare Route Data`
- **Version-specific requirements:**  
  Uses Switch node `typeVersion 3.2`.
- **Edge cases or potential failure types:**  
  - No fallback handling shown; unmatched routed items may stop here depending on default behavior
  - Case sensitivity is enabled; `rice` or `sprint` would not match
  - Because all matches lead to the same node, route-specific differentiation is not yet implemented
  - If tags are comma-separated or inconsistent, substring matches may misroute items
- **Sub-workflow reference:**  
  None

### Node: Prepare Route Data
- **Type and technical role:**  
  `n8n-nodes-base.code` — transforms email data into a normalized signal object for insertion into another database.
- **Configuration choices:**  
  Produces:
  - `title`
  - `type` = `Email Intel`
  - `source`
  - `content`
  - `category`
  - `urgency`
  - `customer`
  - `revenueImpact`
  - `tags`
  - `sourceEmailId`
  - `routedFrom` = `Email Router`
- **Key expressions or variables used:**  
  - `$input.item.json`
- **Input and output connections:**  
  - Input: `Switch by Route Destination`
  - Output: `Create in Signal Stream`
- **Version-specific requirements:**  
  Uses Code node `typeVersion 2`.
- **Edge cases or potential failure types:**  
  - No route-specific payload customization despite separate switch conditions
  - Missing summary or category values still produce a record, but with reduced quality
- **Sub-workflow reference:**  
  None

### Node: Create in Signal Stream
- **Type and technical role:**  
  `n8n-nodes-base.notion` — creates a new page in a separate Notion database used as a signal stream.
- **Configuration choices:**  
  - Resource: `databasePage`
  - Database ID: `31e06bab-3ebe-811b-b204-c5f41b273303`
  - Creates properties:
    - `Signal` title = `{{$json.title}}`
    - `Type` select = `{{$json.type}}`
    - `Source` rich text
    - `Content` rich text
    - `Category` select = `{{$json.category}}`
    - `Urgency` select = `{{$json.urgency}}`
    - `Status` select = `New`
- **Key expressions or variables used:**  
  - `={{ $json.title }}`
  - `={{ $json.type }}`
  - `={{ $json.category }}`
  - `={{ $json.urgency }}`
- **Input and output connections:**  
  - Input: `Prepare Route Data`
  - Output: `Update Status - Routed`
- **Version-specific requirements:**  
  Uses Notion node `typeVersion 2.2`.
- **Edge cases or potential failure types:**  
  - `Source` and `Content` property mappings are included but no explicit visible expression values are shown; verify they are actually mapped in n8n
  - Notion select options must exist or be creatable depending on integration behavior
  - Destination database schema must exactly match property names and types
- **Sub-workflow reference:**  
  None

### Node: Update Status - Routed
- **Type and technical role:**  
  `n8n-nodes-base.notion` — updates the original email-intelligence page after routing.
- **Configuration choices:**  
  - Page ID from `Extract Email Data`
  - Sets:
    - `Status` = `Processed`
    - `Processed At` = current timestamp
    - `Action Taken` rich text field present but blank in visible config
- **Key expressions or variables used:**  
  - `={{ $('Extract Email Data').item.json.pageId }}`
  - `={{ $now.toISO() }}`
- **Input and output connections:**  
  - Input: `Create in Signal Stream`
  - Output: none
- **Version-specific requirements:**  
  Uses Notion node `typeVersion 2.2`.
- **Edge cases or potential failure types:**  
  - Source page update may fail after successful signal creation, causing duplicate routing risk on later reruns
- **Sub-workflow reference:**  
  None

---

## 2.6 Archived Path

**Overview:**  
This branch prepares Gmail archive/label modification data and then marks the original item as processed. The workflow currently stops at metadata preparation and does not actually call Gmail to archive the message.

**Nodes Involved:**  
- Prepare Archive Labels
- Update Status - Archived

### Node: Prepare Archive Labels
- **Type and technical role:**  
  `n8n-nodes-base.code` — builds an archive action payload.
- **Configuration choices:**  
  Produces:
  - `emailId`
  - `threadId`
  - `addLabels` = `['PM-Processed', 'Auto-Archived']`
  - `removeLabels` = `['INBOX', 'UNREAD']`
  - `action` = `archive`
  - `subject`
  - `pageId`
- **Key expressions or variables used:**  
  - `$input.item.json`
- **Input and output connections:**  
  - Input: `Switch by Status` (Archived branch)
  - Output: `Update Status - Archived`
- **Version-specific requirements:**  
  Uses Code node `typeVersion 2`.
- **Edge cases or potential failure types:**  
  - This node only prepares data; no Gmail API action is performed
  - Missing `emailId`/`threadId` means a future archive step would not have enough identifiers
  - Custom labels such as `PM-Processed` may not exist in Gmail
- **Sub-workflow reference:**  
  None

### Node: Update Status - Archived
- **Type and technical role:**  
  `n8n-nodes-base.notion` — marks the source page as processed after archive metadata preparation.
- **Configuration choices:**  
  - Page ID from current JSON (`{{$json.pageId}}`)
  - Sets:
    - `Status` = `Processed`
    - `Processed At` = current timestamp
    - `Action Taken` rich text field present but blank in visible config
- **Key expressions or variables used:**  
  - `={{ $json.pageId }}`
  - `={{ $now.toISO() }}`
- **Input and output connections:**  
  - Input: `Prepare Archive Labels`
  - Output: none
- **Version-specific requirements:**  
  Uses Notion node `typeVersion 2.2`.
- **Edge cases or potential failure types:**  
  - Notion will mark the record as processed even though no Gmail archive action has actually occurred
  - If this is used operationally, users may assume archiving happened when it did not
- **Sub-workflow reference:**  
  None

---

## 2.7 Documentation / Visual Annotation Nodes

**Overview:**  
These sticky notes explain intent and visually group the workflow. They do not affect execution but provide important operator context.

**Nodes Involved:**  
- Sticky Note
- Sticky Note - Routes
- Sticky Note1

### Node: Sticky Note
- **Type and technical role:**  
  `n8n-nodes-base.stickyNote` — canvas annotation.
- **Configuration choices:**  
  Content explains the overall workflow behavior when email status is set in Notion.
- **Input and output connections:**  
  None
- **Version-specific requirements:**  
  Sticky note `typeVersion 1`
- **Edge cases or potential failure types:**  
  None
- **Sub-workflow reference:**  
  None

### Node: Sticky Note - Routes
- **Type and technical role:**  
  `n8n-nodes-base.stickyNote` — canvas annotation for routing/action section.
- **Configuration choices:**  
  Content describes the four-way action router.
- **Input and output connections:**  
  None
- **Version-specific requirements:**  
  Sticky note `typeVersion 1`
- **Edge cases or potential failure types:**  
  None
- **Sub-workflow reference:**  
  None

### Node: Sticky Note1
- **Type and technical role:**  
  `n8n-nodes-base.stickyNote` — annotation for the “closed loop” completion pattern.
- **Configuration choices:**  
  Explains that each branch updates the Notion source entry to `Processed`.
- **Input and output connections:**  
  None
- **Version-specific requirements:**  
  Sticky note `typeVersion 1`
- **Edge cases or potential failure types:**  
  None
- **Sub-workflow reference:**  
  None

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Poll Email Intelligence | Notion | Read candidate email-intelligence pages from the source Notion database |  | Extract Email Data | ## How it works When you set an email's status in Notion (Responded, Delegated, Routed, or Archived), this workflow executes the action: sends the reply, forwards to a delegate, routes to Jira/backlog, or archives in Gmail. |
| Extract Email Data | Code | Flatten Notion page data into normalized email fields | Poll Email Intelligence | Switch by Status | ## Action Router 4-way switch: Responded sends the draft reply via Gmail thread, Delegated forwards + Slack DMs the delegate, Routed creates items in destination DBs, Archived applies Gmail labels. |
| Switch by Status | Switch | Route each item based on Notion status | Extract Email Data | Gmail Send Reply; Gmail Forward; Switch by Route Destination; Prepare Archive Labels | ## Action Router 4-way switch: Responded sends the draft reply via Gmail thread, Delegated forwards + Slack DMs the delegate, Routed creates items in destination DBs, Archived applies Gmail labels. |
| Gmail Send Reply | Gmail | Send a response email using the prepared draft | Switch by Status | Update Status - Responded | ## Action Router 4-way switch: Responded sends the draft reply via Gmail thread, Delegated forwards + Slack DMs the delegate, Routed creates items in destination DBs, Archived applies Gmail labels. |
| Update Status - Responded | Notion | Mark replied item as processed in Notion | Gmail Send Reply |  | ## Closed Loop After each action, updates the Email Intelligence entry in Notion to 'Processed' with a timestamp and reference. |
| Gmail Forward | Gmail | Send a contextual forward/delegation email | Switch by Status | Slack DM Delegate | ## Action Router 4-way switch: Responded sends the draft reply via Gmail thread, Delegated forwards + Slack DMs the delegate, Routed creates items in destination DBs, Archived applies Gmail labels. |
| Slack DM Delegate | Slack | Notify delegate in Slack with summary/context | Gmail Forward | Update Status - Delegated | ## Closed Loop After each action, updates the Email Intelligence entry in Notion to 'Processed' with a timestamp and reference. |
| Update Status - Delegated | Notion | Mark delegated item as processed in Notion | Slack DM Delegate |  | ## Closed Loop After each action, updates the Email Intelligence entry in Notion to 'Processed' with a timestamp and reference. |
| Switch by Route Destination | Switch | Determine routed destination based on tags/category heuristics | Switch by Status | Prepare Route Data | ## Action Router 4-way switch: Responded sends the draft reply via Gmail thread, Delegated forwards + Slack DMs the delegate, Routed creates items in destination DBs, Archived applies Gmail labels. |
| Prepare Route Data | Code | Convert email record into normalized signal payload | Switch by Route Destination | Create in Signal Stream | ## Closed Loop After each action, updates the Email Intelligence entry in Notion to 'Processed' with a timestamp and reference. |
| Create in Signal Stream | Notion | Create a signal page in the destination Notion database | Prepare Route Data | Update Status - Routed | ## Closed Loop After each action, updates the Email Intelligence entry in Notion to 'Processed' with a timestamp and reference. |
| Update Status - Routed | Notion | Mark routed item as processed in Notion | Create in Signal Stream |  | ## Closed Loop After each action, updates the Email Intelligence entry in Notion to 'Processed' with a timestamp and reference. |
| Prepare Archive Labels | Code | Build metadata for Gmail archive/label changes | Switch by Status | Update Status - Archived | ## Action Router 4-way switch: Responded sends the draft reply via Gmail thread, Delegated forwards + Slack DMs the delegate, Routed creates items in destination DBs, Archived applies Gmail labels. |
| Update Status - Archived | Notion | Mark archived item as processed in Notion | Prepare Archive Labels |  | ## Closed Loop After each action, updates the Email Intelligence entry in Notion to 'Processed' with a timestamp and reference. |
| Sticky Note | Sticky Note | Visual explanation of workflow purpose |  |  | ## How it works When you set an email's status in Notion (Responded, Delegated, Routed, or Archived), this workflow executes the action: sends the reply, forwards to a delegate, routes to Jira/backlog, or archives in Gmail. |
| Sticky Note - Routes | Sticky Note | Visual annotation of routing logic |  |  | ## Action Router 4-way switch: Responded sends the draft reply via Gmail thread, Delegated forwards + Slack DMs the delegate, Routed creates items in destination DBs, Archived applies Gmail labels. |
| Sticky Note1 | Sticky Note | Visual annotation of post-action update pattern |  |  | ## Closed Loop After each action, updates the Email Intelligence entry in Notion to 'Processed' with a timestamp and reference. |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it **Email Router**.
   - Optionally add tags similar to:
     - `Email Routing`
     - `PM-DailyOS`

2. **Add a Notion node named `Poll Email Intelligence`**
   - Type: **Notion**
   - Resource: **Database Page**
   - Operation: **Get Many / Get All**
   - Database ID: `31e06bab-3ebe-81d6-8c12-f607c8ff3b3f`
   - Limit: `10`
   - Filter type: `Formula`
   - Configure a Notion credential with access to the source database.
   - Important: ensure the integration has permission to the database.

3. **Add a Code node named `Extract Email Data`**
   - Connect `Poll Email Intelligence` → `Extract Email Data`
   - Paste logic that extracts and normalizes Notion properties into a flat JSON object with the following keys:
     - `pageId`
     - `notionUrl`
     - `subject`
     - `from`
     - `category`
     - `urgency`
     - `status`
     - `summary`
     - `keyAsk`
     - `draftResponse`
     - `customer`
     - `revenueImpact`
     - `suggestedAction`
     - `delegateTo`
     - `emailId`
     - `threadId`
     - `tags`
   - Use safe optional chaining and default values for missing properties.

4. **Add a Switch node named `Switch by Status`**
   - Connect `Extract Email Data` → `Switch by Status`
   - Configure four outputs with exact string equality checks on `{{$json.status}}`:
     1. `Responded`
     2. `Delegated`
     3. `Routed`
     4. `Archived`
   - Set fallback output to **none** if you want unmatched items to stop.

5. **Build the Responded branch**
   1. Add a **Gmail** node named `Gmail Send Reply`
      - Connect from output 1 of `Switch by Status` corresponding to `Responded`
      - Subject: `Re: {{$json.subject}}`
      - Message/body: `{{$json.draftResponse}}`
      - Configure Gmail OAuth2 credentials
      - If you want real thread replies, explicitly configure thread/message parameters if supported by your Gmail node version
   2. Add a **Notion** node named `Update Status - Responded`
      - Connect `Gmail Send Reply` → `Update Status - Responded`
      - Resource: `Database Page`
      - Operation: `Update`
      - Page ID: `{{$('Extract Email Data').item.json.pageId}}`
      - Set properties:
        - `Status` = `Processed`
        - `Processed At` = `{{$now.toISO()}}`
        - `Action Taken` = optional explanatory text such as “Reply sent via Gmail”
      - Use the same Notion credential as the source database if permitted.

6. **Build the Delegated branch**
   1. Add a **Gmail** node named `Gmail Forward`
      - Connect from output 2 of `Switch by Status` corresponding to `Delegated`
      - Subject: `FWD: {{$json.subject}}`
      - Body: compose a forwarding message containing summary, ask, urgency, customer, and revenue impact
      - Configure the recipient explicitly if you want operational reliability; the provided JSON does not show a visible recipient field
      - Gmail OAuth2 credential required
   2. Add a **Slack** node named `Slack DM Delegate`
      - Connect `Gmail Forward` → `Slack DM Delegate`
      - Configure message text with:
        - subject
        - sender
        - urgency
        - summary
        - key ask
      - Configure Slack credentials
      - Explicitly define target user/channel/DM behavior; the imported workflow does not expose a visible recipient mapping
   3. Add a **Notion** node named `Update Status - Delegated`
      - Connect `Slack DM Delegate` → `Update Status - Delegated`
      - Update the source page:
        - `Status` = `Processed`
        - `Processed At` = `{{$now.toISO()}}`
        - `Action Taken` = optional text such as “Delegated by Gmail + Slack”

7. **Build the Routed branch**
   1. Add a **Switch** node named `Switch by Route Destination`
      - Connect from output 3 of `Switch by Status` corresponding to `Routed`
      - Add these rules:
        1. `{{$json.tags}}` contains `RICE`
        2. `{{$json.category}}` contains `Customer`
        3. `{{$json.tags}}` contains `Sprint`
        4. `{{$json.category}}` equals `Competitor Intel`
      - Current design sends all outputs to the same next node
   2. Add a **Code** node named `Prepare Route Data`
      - Connect all outputs of `Switch by Route Destination` → `Prepare Route Data`
      - Build this normalized payload:
        - `title` = subject
        - `type` = `Email Intel`
        - `source` = from
        - `content` = summary
        - `category`
        - `urgency`
        - `customer`
        - `revenueImpact`
        - `tags`
        - `sourceEmailId`
        - `routedFrom` = `Email Router`
   3. Add a **Notion** node named `Create in Signal Stream`
      - Connect `Prepare Route Data` → `Create in Signal Stream`
      - Resource: `Database Page`
      - Operation: `Create`
      - Destination database ID: `31e06bab-3ebe-811b-b204-c5f41b273303`
      - Map properties:
        - `Signal` (title) = `{{$json.title}}`
        - `Type` (select) = `{{$json.type}}`
        - `Source` (rich text) = `{{$json.source}}`
        - `Content` (rich text) = `{{$json.content}}`
        - `Category` (select) = `{{$json.category}}`
        - `Urgency` (select) = `{{$json.urgency}}`
        - `Status` (select) = `New`
      - Ensure the destination Notion database has matching property names/types.
   4. Add a **Notion** node named `Update Status - Routed`
      - Connect `Create in Signal Stream` → `Update Status - Routed`
      - Update source page:
        - `Status` = `Processed`
        - `Processed At` = `{{$now.toISO()}}`
        - `Action Taken` = optional text such as “Routed to Signal Stream”

8. **Build the Archived branch**
   1. Add a **Code** node named `Prepare Archive Labels`
      - Connect from output 4 of `Switch by Status` corresponding to `Archived`
      - Produce:
        - `emailId`
        - `threadId`
        - `addLabels` = `PM-Processed`, `Auto-Archived`
        - `removeLabels` = `INBOX`, `UNREAD`
        - `action` = `archive`
        - `subject`
        - `pageId`
   2. Add a **Notion** node named `Update Status - Archived`
      - Connect `Prepare Archive Labels` → `Update Status - Archived`
      - Update source page:
        - `Status` = `Processed`
        - `Processed At` = `{{$now.toISO()}}`
        - `Action Taken` = optional text such as “Archive metadata prepared”
   3. **Optional but recommended:** add a real Gmail archive/modify node between these two nodes
      - Use Gmail API or Gmail node support for label modification
      - Remove `INBOX`, optionally remove `UNREAD`
      - Add labels if they exist
      - Only after successful Gmail modification should you mark Notion as processed

9. **Add sticky notes for operational clarity**
   - Add a sticky note titled conceptually **How it works**
   - Add a sticky note for **Action Router**
   - Add a sticky note for **Closed Loop**
   - These are optional for execution but useful for maintainability.

10. **Credential setup**
    - **Notion credential**
      - Must have access to both:
        - Source Email Intelligence database
        - Destination Signal Stream database
    - **Gmail OAuth2 credential**
      - Required for `Gmail Send Reply`
      - Required for `Gmail Forward`
      - If implementing real archiving, also required for Gmail modification node
    - **Slack credential**
      - Required for `Slack DM Delegate`
      - Must include scopes for posting messages and, if DMing users, user lookup/open-conversation scopes as needed

11. **Schema requirements in the source Notion database**
    - Ensure these properties exist with compatible types:
      - `Subject` — Title
      - `From` — Rich text
      - `Category` — Select
      - `Urgency` — Select
      - `Status` — Select
      - `Summary` — Rich text
      - `Key Ask` — Rich text
      - `Draft Response` — Rich text
      - `Customer` — Rich text
      - `Revenue Impact` — Rich text
      - `Suggested Action` — Select
      - `Delegate To` — Rich text
      - `Email ID` — Rich text
      - `Thread ID` — Rich text
      - `Tags` — Rich text
      - `Processed At` — Date
      - `Action Taken` — Rich text

12. **Schema requirements in the destination Notion database**
    - Ensure these properties exist:
      - `Signal` — Title
      - `Type` — Select
      - `Source` — Rich text
      - `Content` — Rich text
      - `Category` — Select
      - `Urgency` — Select
      - `Status` — Select

13. **Operational validation checklist**
    - Test one record for each status value:
      - `Responded`
      - `Delegated`
      - `Routed`
      - `Archived`
    - Verify exact status spelling
    - Confirm Gmail nodes have valid recipients and/or thread behavior
    - Confirm Slack reaches the intended recipient
    - Confirm Notion updates happen only after successful external actions
    - Add error handling or retries if this will be used in production

14. **Recommended production improvements**
    - Replace polling start with a proper trigger or scheduled trigger
    - Filter Notion pages to only unprocessed actionable statuses
    - Add an idempotency guard so already-processed pages are skipped
    - Populate `Action Taken` with explicit audit text
    - Implement real Gmail archive/label modification
    - Differentiate route destinations instead of sending every route to the same destination node
    - Use `delegateTo` to dynamically select Gmail recipient and Slack recipient

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| When you set an email's status in Notion (Responded, Delegated, Routed, or Archived), this workflow executes the action: sends the reply, forwards to a delegate, routes to Jira/backlog, or archives in Gmail. | Workflow canvas note |
| 4-way switch: Responded sends the draft reply via Gmail thread, Delegated forwards + Slack DMs the delegate, Routed creates items in destination DBs, Archived applies Gmail labels. | Workflow canvas note |
| After each action, updates the Email Intelligence entry in Notion to “Processed” with a timestamp and reference. | Workflow canvas note |
| The workflow title provided by the user is “Route email actions from Notion with Gmail, Slack, and Jira”, but the current JSON implementation routes to Notion Signal Stream rather than Jira. | Important implementation note |
| The archive branch currently prepares label/archive data but does not perform the Gmail archive action. | Important implementation note |
| The workflow is inactive (`active: false`) in the provided export. | Deployment note |