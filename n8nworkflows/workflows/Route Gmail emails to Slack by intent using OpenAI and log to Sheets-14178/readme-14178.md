Route Gmail emails to Slack by intent using OpenAI and log to Sheets

https://n8nworkflows.xyz/workflows/route-gmail-emails-to-slack-by-intent-using-openai-and-log-to-sheets-14178


# Route Gmail emails to Slack by intent using OpenAI and log to Sheets

# 1. Workflow Overview

This workflow monitors a Gmail inbox, uses OpenAI to classify each incoming email by intent, routes non-spam messages to the appropriate Slack channel, and logs all processed emails to Google Sheets.

Typical use cases include:

- Shared inbox triage for small teams
- Automatic routing of sales, support, and billing emails
- Lightweight audit logging of email intake and AI classification
- Reducing manual inbox review effort

The workflow is organized into the following logical blocks.

## 1.1 Input Reception

A Gmail trigger polls for new emails every minute and starts the workflow whenever a new message is detected.

## 1.2 Settings and Email Field Extraction

A Set node centralizes configuration values such as Slack channel IDs and the Google Sheet ID, while also extracting normalized email fields like sender, subject, and body/snippet.

## 1.3 AI Classification

An OpenAI node sends the extracted email content to a model with a structured prompt. The model must return JSON containing category, summary, and priority.

## 1.4 Result Parsing and Normalization

A Code node parses the AI response, handles alternate response shapes, and applies fallback defaults if parsing fails.

## 1.5 Routing to Slack

A Switch node routes emails by category to one of three Slack nodes: sales, support, or billing. Any uncaptured category falls into the fallback output and is effectively dropped because nothing is connected there.

## 1.6 Logging to Google Sheets

In parallel with Slack routing, the processed email is appended to a Google Sheet for traceability and later review.

---

# 2. Block-by-Block Analysis

## 2.1 Input Reception

**Overview:**  
This block detects new Gmail messages and acts as the workflow entry point. It is the only trigger in the workflow and starts execution on a polling schedule.

**Nodes Involved:**  
- Watch for New Emails

### Node Details

#### Watch for New Emails
- **Type and technical role:** `n8n-nodes-base.gmailTrigger`  
  Polling trigger for new Gmail messages.
- **Configuration choices:**
  - Polling mode is set to run **every minute**
  - `simple: false`, meaning the node is configured to return the fuller Gmail payload rather than a simplified subset
  - No Gmail filters are defined, so it watches broadly for new emails available to the connected account
- **Key expressions or variables used:**  
  None in the node itself.
- **Input and output connections:**
  - Input: none, as this is the trigger
  - Output: `Configure Settings`
- **Version-specific requirements:**  
  Uses node version **1.2**
- **Edge cases or potential failure types:**
  - Gmail OAuth credential issues or expired tokens
  - Polling delays or missed events if Gmail/API limits are hit
  - Unexpected payload differences depending on mailbox/message format
  - If the connected Gmail account lacks the expected access scope, the trigger may fail to start
- **Sub-workflow reference:**  
  None

---

## 2.2 Settings and Email Field Extraction

**Overview:**  
This block prepares reusable configuration and standardizes the email data that downstream nodes consume. It combines static settings with expressions derived from the Gmail payload.

**Nodes Involved:**  
- Configure Settings

### Node Details

#### Configure Settings
- **Type and technical role:** `n8n-nodes-base.set`  
  Creates and reshapes fields for downstream processing.
- **Configuration choices:**
  - Defines static placeholders for:
    - `slackChannel_sales`
    - `slackChannel_support`
    - `slackChannel_billing`
    - `googleSheetId`
  - Extracts normalized message fields:
    - `emailFrom`
    - `emailSubject`
    - `emailBody`
- **Key expressions or variables used:**
  - `={{ $json.from?.emailAddress || $json.from }}`
    - Attempts to support either an object-based sender field or a plain string sender field
  - `={{ $json.subject }}`
  - `={{ $json.snippet || $json.text || '' }}`
    - Prefers Gmail snippet, otherwise uses text, otherwise empty string
- **Input and output connections:**
  - Input: `Watch for New Emails`
  - Output: `Classify Email Intent`
- **Version-specific requirements:**  
  Uses node version **3.4**
- **Edge cases or potential failure types:**
  - If Gmail returns sender data in an unexpected shape, `emailFrom` may be blank or malformed
  - `snippet` may be truncated and may not contain enough context for high-quality classification
  - The configured Slack channel IDs are stored here but are not actually consumed by the Slack nodes in the current design
  - `googleSheetId` is used later; if left as the placeholder value, Google Sheets logging will fail
- **Sub-workflow reference:**  
  None

---

## 2.3 AI Classification

**Overview:**  
This block sends the extracted email content to OpenAI for classification. The prompt enforces a strict JSON output containing category, summary, and priority.

**Nodes Involved:**  
- Classify Email Intent

### Node Details

#### Classify Email Intent
- **Type and technical role:** `@n8n/n8n-nodes-langchain.openAi`  
  LLM interaction node used to classify email intent and produce structured metadata.
- **Configuration choices:**
  - Model: `gpt-4o-mini`
  - Temperature: `0.1` for low-variance, more deterministic output
  - Max tokens: `150`
  - JSON output enabled
  - System prompt defines exactly four valid categories:
    - sales
    - support
    - billing
    - spam
  - The system message explicitly requires:
    - JSON only
    - `category`
    - `summary`
    - `priority`
- **Key expressions or variables used:**
  - User message body:
    - `From: {{ $json.emailFrom }}`
    - `Subject: {{ $json.emailSubject }}`
    - `Body: {{ $json.emailBody }}`
- **Input and output connections:**
  - Input: `Configure Settings`
  - Output: `Parse AI Response`
- **Version-specific requirements:**  
  Uses node version **1.8**
- **Edge cases or potential failure types:**
  - OpenAI credential or quota errors
  - Model may still return malformed JSON despite instructions
  - Truncated or low-context email body can reduce classification accuracy
  - Priority values may drift outside `low|medium|high` if the model ignores instructions
  - If the LangChain/OpenAI node output format changes, downstream parsing may need adjustment
- **Sub-workflow reference:**  
  None

---

## 2.4 Result Parsing and Normalization

**Overview:**  
This block converts the AI output into a reliable internal structure. It also merges back the configuration fields and adds a processing timestamp.

**Nodes Involved:**  
- Parse AI Response

### Node Details

#### Parse AI Response
- **Type and technical role:** `n8n-nodes-base.code`  
  JavaScript transformation node that parses AI output and normalizes final fields.
- **Configuration choices:**
  - Reads the original configuration from the `Configure Settings` node using node reference lookup
  - Reads the first incoming AI response item
  - Supports multiple possible AI response shapes:
    - direct object with `category`
    - `text`
    - `message.content`
    - `choices[0].message.content`
    - fallback stringified payload
  - If parsing fails, defaults to:
    - `category: support`
    - `summary: Could not classify`
    - `priority: medium`
  - Adds `timestamp` using current ISO datetime
- **Key expressions or variables used:**
  - `$('Configure Settings').first().json`
  - `$input.first().json`
  - `new Date().toISOString()`
- **Input and output connections:**
  - Input: `Classify Email Intent`
  - Outputs:
    - `Route by Category`
    - `Log to Google Sheet`
- **Version-specific requirements:**  
  Uses node version **2**
- **Edge cases or potential failure types:**
  - If node execution data from `Configure Settings` is unavailable, the cross-node lookup could fail
  - If the OpenAI node returns a deeply different format, parsing may silently fall back
  - Defaulting parse failures to `support` can create misrouting instead of an explicit exception path
  - If invalid or empty JSON is returned, logging still occurs with fallback values
- **Sub-workflow reference:**  
  None

---

## 2.5 Routing to Slack

**Overview:**  
This block routes classified emails to the correct Slack destination. It handles sales, support, and billing explicitly, while uncaptured values such as spam fall through to an unconnected fallback.

**Nodes Involved:**  
- Route by Category
- Send to Sales Channel
- Send to Support Channel
- Send to Billing Channel

### Node Details

#### Route by Category
- **Type and technical role:** `n8n-nodes-base.switch`  
  Conditional router that branches based on the normalized `category`.
- **Configuration choices:**
  - Three named outputs:
    - `sales`
    - `support`
    - `billing`
  - Case-insensitive equality checks
  - Fallback output set to `extra`
  - No node is connected to the fallback output
- **Key expressions or variables used:**
  - `={{ $json.category }}`
- **Input and output connections:**
  - Input: `Parse AI Response`
  - Outputs:
    - `Send to Sales Channel`
    - `Send to Support Channel`
    - `Send to Billing Channel`
    - fallback `extra` unused
- **Version-specific requirements:**  
  Uses node version **3.2**
- **Edge cases or potential failure types:**
  - Any category outside the three explicit branches, including `spam`, will not go to Slack
  - Typos or formatting issues in `category` will also fall into the unused fallback output
  - This behavior matches the intended spam-drop design, but other unexpected values are also silently discarded
- **Sub-workflow reference:**  
  None

#### Send to Sales Channel
- **Type and technical role:** `n8n-nodes-base.slack`  
  Sends a formatted Slack message for sales-classified emails.
- **Configuration choices:**
  - Authentication: OAuth2
  - Channel selection mode set to `channel`
  - Channel configured as `#general` in name mode
  - Message includes emoji, priority, sender, subject, and summary
- **Key expressions or variables used:**
  - `{{ $json.priority }}`
  - `{{ $json.emailFrom }}`
  - `{{ $json.emailSubject }}`
  - `{{ $json.summary }}`
- **Input and output connections:**
  - Input: `Route by Category` (sales output)
  - Output: none
- **Version-specific requirements:**  
  Uses node version **2.4**
- **Edge cases or potential failure types:**
  - Slack OAuth scope or auth issues
  - The node does not use `slackChannel_sales` from the Set node, despite the workflow notes suggesting it should
  - If the named channel does not exist or the app cannot post there, the node will fail
  - Special characters from email data may affect Slack formatting
- **Sub-workflow reference:**  
  None

#### Send to Support Channel
- **Type and technical role:** `n8n-nodes-base.slack`  
  Sends a formatted Slack message for support-classified emails.
- **Configuration choices:**
  - Authentication: OAuth2
  - Channel configured as `#general` in name mode
  - Message formatted for support notification
- **Key expressions or variables used:**
  - `{{ $json.priority }}`
  - `{{ $json.emailFrom }}`
  - `{{ $json.emailSubject }}`
  - `{{ $json.summary }}`
- **Input and output connections:**
  - Input: `Route by Category` (support output)
  - Output: none
- **Version-specific requirements:**  
  Uses node version **2.4**
- **Edge cases or potential failure types:**
  - Same Slack credential/channel risks as above
  - The configured `slackChannel_support` field is not used by this node
- **Sub-workflow reference:**  
  None

#### Send to Billing Channel
- **Type and technical role:** `n8n-nodes-base.slack`  
  Sends a formatted Slack message for billing-classified emails.
- **Configuration choices:**
  - Authentication: OAuth2
  - Channel configured as `#general` in name mode
  - Message formatted for billing notification
- **Key expressions or variables used:**
  - `{{ $json.priority }}`
  - `{{ $json.emailFrom }}`
  - `{{ $json.emailSubject }}`
  - `{{ $json.summary }}`
- **Input and output connections:**
  - Input: `Route by Category` (billing output)
  - Output: none
- **Version-specific requirements:**  
  Uses node version **2.4**
- **Edge cases or potential failure types:**
  - Same Slack credential/channel risks as above
  - The configured `slackChannel_billing` field is not used by this node
- **Sub-workflow reference:**  
  None

---

## 2.6 Logging to Google Sheets

**Overview:**  
This block records each processed email and its AI-generated metadata in a spreadsheet. It runs in parallel to Slack routing, so logging does not depend on successful Slack delivery.

**Nodes Involved:**  
- Log to Google Sheet

### Node Details

#### Log to Google Sheet
- **Type and technical role:** `n8n-nodes-base.googleSheets`  
  Appends a row into a Google Sheet.
- **Configuration choices:**
  - Operation: `append`
  - Sheet name: `Sheet1`
  - Document ID is read from `Configure Settings`
  - Column mapping is explicitly defined:
    - From
    - Subject
    - Summary
    - Category
    - Priority
    - Timestamp
- **Key expressions or variables used:**
  - `={{ $json.emailFrom }}`
  - `={{ $json.emailSubject }}`
  - `={{ $json.summary }}`
  - `={{ $json.category }}`
  - `={{ $json.priority }}`
  - `={{ $json.timestamp }}`
  - `={{ $('Configure Settings').first().json.googleSheetId }}`
- **Input and output connections:**
  - Input: `Parse AI Response`
  - Output: none
- **Version-specific requirements:**  
  Uses node version **4.5**
- **Edge cases or potential failure types:**
  - Google Sheets credential problems
  - Invalid document ID or inaccessible spreadsheet
  - `Sheet1` must exist unless the node is changed to target another sheet
  - Column names in the spreadsheet should match the mapping exactly for clean appends
  - Because logging is parallel to routing, Slack failure does not prevent logging, and logging failure does not prevent Slack attempts
- **Sub-workflow reference:**  
  None

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Watch for New Emails | Gmail Trigger | Poll Gmail for new incoming emails every minute |  | Configure Settings | ## 1. New email comes in<br>Gmail trigger polls every minute. You can change this to whatever interval you want. |
| Configure Settings | Set | Store configuration values and normalize email fields | Watch for New Emails | Classify Email Intent | ## 2. Settings and extraction<br>Pulls out the sender, subject, and body. Also holds your Slack channel IDs and Sheet ID. |
| Classify Email Intent | OpenAI (LangChain) | Classify email into category and generate summary/priority | Configure Settings | Parse AI Response | ## 3. AI classifies the email<br>OpenAI reads the email and decides: is this sales, support, billing, or spam? Also gives you a summary and priority level. |
| Parse AI Response | Code | Parse AI JSON output, apply fallback defaults, add timestamp | Classify Email Intent | Route by Category; Log to Google Sheet | ## 4. Route to the right channel<br>Sales emails go to #sales, support goes to #support, billing goes to #billing. Spam gets dropped entirely. |
| Route by Category | Switch | Route items to the correct Slack branch by category | Parse AI Response | Send to Sales Channel; Send to Support Channel; Send to Billing Channel | ## 4. Route to the right channel<br>Sales emails go to #sales, support goes to #support, billing goes to #billing. Spam gets dropped entirely. |
| Send to Sales Channel | Slack | Notify Slack for sales emails | Route by Category |  | ## 4. Route to the right channel<br>Sales emails go to #sales, support goes to #support, billing goes to #billing. Spam gets dropped entirely. |
| Send to Support Channel | Slack | Notify Slack for support emails | Route by Category |  | ## 4. Route to the right channel<br>Sales emails go to #sales, support goes to #support, billing goes to #billing. Spam gets dropped entirely. |
| Send to Billing Channel | Slack | Notify Slack for billing emails | Route by Category |  | ## 4. Route to the right channel<br>Sales emails go to #sales, support goes to #support, billing goes to #billing. Spam gets dropped entirely. |
| Log to Google Sheet | Google Sheets | Append processed email details to a spreadsheet | Parse AI Response |  | ## 5. Log everything<br>Every email gets recorded to Google Sheets with the AI's classification. Handy for reporting and catching anything the AI gets wrong. |
| Sticky Note | Sticky Note | Visual documentation for workflow purpose, setup, and customization |  |  | ### Route incoming emails to Slack channels by intent using OpenAI<br><br>If your team is drowning in a shared inbox and someone has to manually read every email to figure out where it should go, this takes that job off your plate. Emails come in, AI reads them, and the right Slack channel gets notified.<br><br>### How it works<br>1. Gmail checks for new emails every minute<br>2. Each email gets sent to OpenAI with a prompt that classifies it as sales, support, billing, or spam<br>3. The AI also writes a one-line summary and assigns a priority (low/medium/high)<br>4. A Switch node routes the message to the right Slack channel. Spam gets quietly dropped.<br>5. Every email is logged to a Google Sheet so you have a full record of what came in and how it was categorized<br><br>### Setup<br>1. Connect your Gmail, OpenAI, Slack, and Google Sheets credentials in n8n<br>2. Create a Google Sheet with columns: Timestamp, From, Subject, Category, Priority, Summary<br>3. Open "Configure Settings" and add your Slack channel IDs and Sheet ID<br>4. Activate the workflow and let it run<br><br>### Customization<br>- Want more categories? Edit the system prompt in the "Classify Email Intent" node and add a new output to the Switch node<br>- Change how often it checks by editing the Gmail trigger (every 5 minutes instead of every minute, for example)<br>- Swap gpt-4o-mini for a different model if you need more accuracy |
| Sticky Note1 | Sticky Note | Visual annotation for trigger block |  |  | ## 1. New email comes in<br>Gmail trigger polls every minute. You can change this to whatever interval you want. |
| Sticky Note2 | Sticky Note | Visual annotation for settings block |  |  | ## 2. Settings and extraction<br>Pulls out the sender, subject, and body. Also holds your Slack channel IDs and Sheet ID. |
| Sticky Note3 | Sticky Note | Visual annotation for AI classification block |  |  | ## 3. AI classifies the email<br>OpenAI reads the email and decides: is this sales, support, billing, or spam? Also gives you a summary and priority level. |
| Sticky Note4 | Sticky Note | Visual annotation for routing block |  |  | ## 4. Route to the right channel<br>Sales emails go to #sales, support goes to #support, billing goes to #billing. Spam gets dropped entirely. |
| Sticky Note5 | Sticky Note | Visual annotation for logging block |  |  | ## 5. Log everything<br>Every email gets recorded to Google Sheets with the AI's classification. Handy for reporting and catching anything the AI gets wrong. |

---

# 4. Reproducing the Workflow from Scratch

Follow these steps to rebuild the workflow manually in n8n.

## 1. Create the Gmail trigger
1. Add a **Gmail Trigger** node.
2. Name it **Watch for New Emails**.
3. Set polling to **Every Minute**.
4. Leave filters empty unless you want to limit which emails are processed.
5. Set `simple` to **false** if you want fuller Gmail payloads like in this workflow.
6. Connect Gmail OAuth credentials with permission to read mailbox events/messages.

## 2. Add the configuration and extraction node
1. Add a **Set** node after the Gmail trigger.
2. Name it **Configure Settings**.
3. Create the following fields:
   - `slackChannel_sales` = `YOUR_SALES_CHANNEL_ID`
   - `slackChannel_support` = `YOUR_SUPPORT_CHANNEL_ID`
   - `slackChannel_billing` = `YOUR_BILLING_CHANNEL_ID`
   - `googleSheetId` = `YOUR_GOOGLE_SHEET_ID`
   - `emailFrom` = `{{ $json.from?.emailAddress || $json.from }}`
   - `emailSubject` = `{{ $json.subject }}`
   - `emailBody` = `{{ $json.snippet || $json.text || '' }}`
4. Connect **Watch for New Emails → Configure Settings**.

## 3. Add the OpenAI classification node
1. Add an **OpenAI** node from the LangChain/OpenAI integration.
2. Name it **Classify Email Intent**.
3. Select model **gpt-4o-mini**.
4. Set:
   - **Temperature** = `0.1`
   - **Max Tokens** = `150`
   - **JSON output** = enabled
5. Configure messages:
   - **System message**:
     ```
     You are an email classifier. Classify the following email into exactly one category: sales, support, billing, or spam.

     Rules:
     - sales: inquiries about pricing, services, partnerships, new business
     - support: help requests, bug reports, technical issues, feature questions
     - billing: invoices, payments, refunds, subscription changes
     - spam: promotional, unsolicited, irrelevant

     Respond with a JSON object only. No other text.
     {"category": "<category>", "summary": "<one sentence summary of the email>", "priority": "<low|medium|high>"}
     ```
   - **User message**:
     ```
     From: {{ $json.emailFrom }}
     Subject: {{ $json.emailSubject }}
     Body: {{ $json.emailBody }}
     ```
6. Connect OpenAI credentials.
7. Connect **Configure Settings → Classify Email Intent**.

## 4. Add the parsing and normalization code node
1. Add a **Code** node after the OpenAI node.
2. Name it **Parse AI Response**.
3. Use JavaScript mode.
4. Paste this logic conceptually:
   - Read JSON from **Configure Settings**
   - Read the first AI response item
   - If the AI response already contains `category`, use it directly
   - Otherwise try to parse one of:
     - `text`
     - `message.content`
     - `choices[0].message.content`
     - stringified whole response
   - If parsing fails, fallback to support/default values
   - Return merged config + classification + timestamp

5. Use the following code:
   ```javascript
   const config = $('Configure Settings').first().json;
   const aiResponse = $input.first().json;

   let parsed;
   try {
     if (aiResponse.category) {
       parsed = aiResponse;
     } else {
       const content = aiResponse.text || aiResponse.message?.content || aiResponse.choices?.[0]?.message?.content || JSON.stringify(aiResponse);
       parsed = typeof content === 'string' ? JSON.parse(content) : content;
     }
   } catch (e) {
     parsed = { category: 'support', summary: 'Could not classify', priority: 'medium' };
   }

   return [{
     json: {
       ...config,
       category: parsed.category || 'support',
       summary: parsed.summary || 'No summary available',
       priority: parsed.priority || 'medium',
       timestamp: new Date().toISOString()
     }
   }];
   ```
6. Connect **Classify Email Intent → Parse AI Response**.

## 5. Add the category router
1. Add a **Switch** node.
2. Name it **Route by Category**.
3. Create three rules with renamed outputs:
   - Output `sales` when `{{ $json.category }}` equals `sales`
   - Output `support` when `{{ $json.category }}` equals `support`
   - Output `billing` when `{{ $json.category }}` equals `billing`
4. Set comparison to **case-insensitive**.
5. Set fallback output to **extra**.
6. Do not connect the fallback if you want spam and unknown categories to be dropped silently.
7. Connect **Parse AI Response → Route by Category**.

## 6. Add the Slack node for sales
1. Add a **Slack** node.
2. Name it **Send to Sales Channel**.
3. Choose OAuth2 authentication.
4. Set action to send a message to a channel.
5. Configure the message text:
   ```text
   :envelope: *New Sales Inquiry* ({{ $json.priority }} priority)
   *From:* {{ $json.emailFrom }}
   *Subject:* {{ $json.emailSubject }}
   *Summary:* {{ $json.summary }}
   ```
6. Set channel selection to a specific channel. In the source workflow it is `#general`.
7. Connect Slack credentials.
8. Connect the **sales** output of **Route by Category** to this node.

## 7. Add the Slack node for support
1. Add another **Slack** node.
2. Name it **Send to Support Channel**.
3. Use OAuth2 authentication.
4. Configure the message:
   ```text
   :wrench: *New Support Request* ({{ $json.priority }} priority)
   *From:* {{ $json.emailFrom }}
   *Subject:* {{ $json.emailSubject }}
   *Summary:* {{ $json.summary }}
   ```
5. Set the destination channel. In the source workflow it is also `#general`.
6. Connect the **support** output of **Route by Category** to this node.

## 8. Add the Slack node for billing
1. Add another **Slack** node.
2. Name it **Send to Billing Channel**.
3. Use OAuth2 authentication.
4. Configure the message:
   ```text
   :receipt: *New Billing Email* ({{ $json.priority }} priority)
   *From:* {{ $json.emailFrom }}
   *Subject:* {{ $json.emailSubject }}
   *Summary:* {{ $json.summary }}
   ```
5. Set the destination channel. In the source workflow it is also `#general`.
6. Connect the **billing** output of **Route by Category** to this node.

## 9. Add Google Sheets logging
1. Add a **Google Sheets** node.
2. Name it **Log to Google Sheet**.
3. Set operation to **Append**.
4. Select or enter the spreadsheet document ID using:
   - `{{ $('Configure Settings').first().json.googleSheetId }}`
5. Set sheet name to **Sheet1**.
6. Use manual column mapping and define:
   - `From` = `{{ $json.emailFrom }}`
   - `Subject` = `{{ $json.emailSubject }}`
   - `Summary` = `{{ $json.summary }}`
   - `Category` = `{{ $json.category }}`
   - `Priority` = `{{ $json.priority }}`
   - `Timestamp` = `{{ $json.timestamp }}`
7. Connect Google Sheets credentials.
8. Connect **Parse AI Response → Log to Google Sheet** directly so logging happens in parallel with Slack routing.

## 10. Prepare external services
1. **Gmail**
   - Connect a Gmail account with access to the mailbox to monitor.
2. **OpenAI**
   - Add an API credential with access to the chosen model.
3. **Slack**
   - Connect a Slack app via OAuth2 with permission to post messages to the selected channels.
4. **Google Sheets**
   - Connect a Google account with edit access to the target spreadsheet.

## 11. Prepare the spreadsheet
1. Create a Google Sheet.
2. Create a worksheet named **Sheet1**.
3. Add these columns in the header row:
   - `Timestamp`
   - `From`
   - `Subject`
   - `Category`
   - `Priority`
   - `Summary`

## 12. Test and activate
1. Send sample emails representing:
   - sales
   - support
   - billing
   - spam
2. Confirm:
   - sales/support/billing messages appear in Slack
   - spam does not get posted to Slack
   - all processed items are appended to the Google Sheet
3. Replace placeholders in **Configure Settings** if needed.
4. Activate the workflow.

## Important implementation note
The current workflow stores Slack channel IDs in **Configure Settings**, but the three Slack nodes do not actually reference those fields. They are hardcoded to channel name `#general`.

If you want the workflow to behave according to its own setup note, update each Slack node’s channel field to use expressions such as:
- Sales: `{{ $json.slackChannel_sales }}`
- Support: `{{ $json.slackChannel_support }}`
- Billing: `{{ $json.slackChannel_billing }}`

Also ensure the Slack node field is configured in a mode compatible with channel IDs if you use IDs rather than names.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Route incoming emails to Slack channels by intent using OpenAI | Overall workflow purpose |
| Gmail checks for new emails every minute | Workflow behavior |
| OpenAI classifies each email as sales, support, billing, or spam | AI processing logic |
| AI also generates a one-line summary and assigns low/medium/high priority | AI output design |
| Spam is dropped from Slack routing via the Switch fallback behavior | Routing design |
| Google Sheets keeps a full record of what came in and how it was categorized | Logging design |
| Setup note: connect Gmail, OpenAI, Slack, and Google Sheets credentials in n8n | Environment preparation |
| Setup note: create a Google Sheet with columns Timestamp, From, Subject, Category, Priority, Summary | Spreadsheet preparation |
| Setup note: open “Configure Settings” and add your Slack channel IDs and Sheet ID | Configuration step |
| Customization note: add more categories by editing the system prompt and adding outputs to the Switch node | Workflow extension |
| Customization note: adjust Gmail polling frequency as needed | Trigger tuning |
| Customization note: replace gpt-4o-mini with another model for different speed/accuracy tradeoffs | Model customization |