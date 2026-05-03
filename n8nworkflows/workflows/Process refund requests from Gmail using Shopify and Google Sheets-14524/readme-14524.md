Process refund requests from Gmail using Shopify and Google Sheets

https://n8nworkflows.xyz/workflows/process-refund-requests-from-gmail-using-shopify-and-google-sheets-14524


# Process refund requests from Gmail using Shopify and Google Sheets

# 1. Workflow Overview

This workflow automates the handling of refund request emails received in Gmail. It monitors unread emails whose subject matches **“Refund Request”**, uses an AI model to extract the **order ID** and a short **refund reason**, retrieves the corresponding order from **Shopify**, checks whether the order status is eligible for automatic approval, sends the appropriate email notifications, and appends an audit log to **Google Sheets**.

Its main use cases are:

- Reducing manual effort for simple refund requests
- Auto-approving requests when order status is suitable
- Escalating ambiguous or ineligible cases to a human team
- Maintaining a structured log of outcomes for reporting or compliance

## 1.1 Input Reception and AI Extraction

The workflow starts from Gmail, filtering incoming unread refund emails. The email subject and snippet are passed to an AI Agent backed by a Groq chat model, which is instructed to return only JSON containing the order ID and a short reason.

## 1.2 Parsed Data Normalization

Because the AI Agent returns a stringified JSON response, a JavaScript Code node parses that string and exposes clean n8n fields such as `order_id` and `reason` for downstream processing.

## 1.3 Shopify Validation

The extracted `order_id` is used to fetch the corresponding order from Shopify. The returned Shopify order data provides the `Status` field used by the decision logic.

## 1.4 Decision Logic

An If node checks whether the Shopify order’s `Status` equals `Delivered`. If true, the refund is auto-approved. Otherwise, the request is routed for manual review.

## 1.5 Notifications and Logging

Depending on the decision path, the workflow sends one or more Gmail messages:
- Auto-approval email to the customer
- Manual-review alert to the internal team
- Follow-up email to the customer if manual review is needed

Both branches end by appending a structured log row to a Google Sheet.

---

# 2. Block-by-Block Analysis

## Block 1: Input Reception and AI Extraction

### Overview

This block detects refund request emails in Gmail and extracts key structured data from natural language content. It converts an unstructured email into machine-usable fields required for Shopify lookup and downstream routing.

### Nodes Involved

- Gmail Trigger
- Groq Chat Model
- AI Agent2

### Node Details

#### 1. Gmail Trigger

- **Type and technical role:** `n8n-nodes-base.gmailTrigger`  
  Polling trigger that checks a Gmail mailbox for new matching messages.
- **Configuration choices:**
  - Query filter: `subject:"Refund Request"`
  - Read status filter: `unread`
  - Polling interval: every minute
- **Key expressions or variables used:**
  - Provides fields later referenced such as:
    - `{{$json.Subject}}`
    - `{{$json.snippet}}`
    - `{{$json.From}}`
- **Input and output connections:**
  - Entry point of the workflow
  - Outputs to **AI Agent2**
- **Version-specific requirements:**
  - Uses node version `1.3`
  - Requires Gmail OAuth2 credentials configured in n8n
- **Edge cases or potential failure types:**
  - Gmail OAuth expiration or revoked consent
  - Query mismatch due to unexpected subject format
  - Duplicate handling issues if unread messages are not later marked/read elsewhere
  - Polling delay due to one-minute schedule
  - Messages with incomplete `snippet` content may reduce extraction quality
- **Sub-workflow reference:** None

#### 2. Groq Chat Model

- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatGroq`  
  Language model provider node used by the AI Agent as its LLM backend.
- **Configuration choices:**
  - Model: `llama-3.3-70b-versatile`
  - No custom options configured
- **Key expressions or variables used:** None directly in this node
- **Input and output connections:**
  - Connected to **AI Agent2** via AI language model port
  - No main data path output used directly
- **Version-specific requirements:**
  - Uses type version `1`
  - Requires a valid Groq API credential
  - Requires n8n environment with LangChain-compatible AI nodes
- **Edge cases or potential failure types:**
  - Invalid API key or quota exhaustion
  - Rate limiting
  - Model availability changes
  - Non-deterministic output despite strict prompting
- **Sub-workflow reference:** None

#### 3. AI Agent2

- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent`  
  AI orchestration node that submits a prompt to the Groq model and returns extracted structured content.
- **Configuration choices:**
  - Prompt type: defined manually
  - Prompt instructs the model to:
    - extract Order ID
    - summarize reason in 5 words or less
    - return only a JSON object
  - Input content:
    - `Subject: {{ $json.Subject }}`
    - `Snippet: {{ $json.snippet }}`
- **Key expressions or variables used:**
  - `{{ $json.Subject }}`
  - `{{ $json.snippet }}`
- **Input and output connections:**
  - Receives email data from **Gmail Trigger**
  - Receives model connection from **Groq Chat Model**
  - Sends main output to **Code in JavaScript**
- **Version-specific requirements:**
  - Uses type version `3`
  - Depends on AI model connection through the `ai_languageModel` input
- **Edge cases or potential failure types:**
  - Model returns text outside JSON despite instruction
  - Missing order ID in the email body/snippet
  - Snippet truncation leads to incomplete extraction
  - Hallucinated order IDs or over-short reasons
- **Sub-workflow reference:** None

---

## Block 2: Parsed Data Normalization

### Overview

This block converts the AI Agent response into actual JSON fields n8n can use reliably. It isolates the extracted `order_id` and `reason` and prepares a backup field.

### Nodes Involved

- Code in JavaScript

### Node Details

#### 1. Code in JavaScript

- **Type and technical role:** `n8n-nodes-base.code`  
  JavaScript transformation node that parses the AI output string into structured JSON.
- **Configuration choices:**
  - Reads `items[0].json.output`
  - Executes `JSON.parse(...)`
  - Returns:
    - `order_id`
    - `reason`
    - `original_summary`
- **Key expressions or variables used:**
  - `items[0].json.output`
  - Returned fields:
    - `cleanData.order_id`
    - `cleanData.reason`
- **Input and output connections:**
  - Input from **AI Agent2**
  - Output to **Get an order**
- **Version-specific requirements:**
  - Uses node version `2`
  - Requires JavaScript execution enabled in n8n
- **Edge cases or potential failure types:**
  - `JSON.parse` throws an error if the AI output is not valid JSON
  - `items[0]` may be undefined if upstream output is empty
  - `order_id` may be missing or malformed
  - `original_summary` duplicates `reason`; harmless but redundant
- **Sub-workflow reference:** None

---

## Block 3: Shopify Validation

### Overview

This block looks up the order in Shopify using the AI-extracted order ID. The returned order data is used to determine refund eligibility.

### Nodes Involved

- Get an order

### Node Details

#### 1. Get an order

- **Type and technical role:** `n8n-nodes-base.shopify`  
  Shopify API node used to fetch an order by ID.
- **Configuration choices:**
  - Operation: `get`
  - Authentication: access token
  - Order ID source: `={{ $json.order_id }}`
- **Key expressions or variables used:**
  - `{{ $json.order_id }}`
- **Input and output connections:**
  - Input from **Code in JavaScript**
  - Output to **If**
- **Version-specific requirements:**
  - Uses type version `1`
  - Requires Shopify credentials using an access token
  - The connected Shopify app/token must have at least `read_orders` permission
- **Edge cases or potential failure types:**
  - Invalid or missing order ID
  - Shopify returns 404 if order does not exist
  - Permissions insufficient to read orders
  - Possible mismatch between extracted 4-digit human order number and Shopify internal order ID expectations, depending on store setup
- **Sub-workflow reference:** None

**Important integration note:**  
This node assumes the extracted `order_id` can be used directly in Shopify’s `get` operation. In many Shopify integrations, there is a difference between:
- internal Shopify order ID
- human-facing order number/name

If emails contain order numbers like `1001`, but the Shopify node expects the internal resource ID, this node may fail or return nothing. In that case, the workflow should be modified to search orders by name/number first.

---

## Block 4: Decision Logic

### Overview

This block decides whether the refund request can be auto-approved. It checks whether the Shopify response contains a `Status` equal to `Delivered`.

### Nodes Involved

- If

### Node Details

#### 1. If

- **Type and technical role:** `n8n-nodes-base.if`  
  Conditional branching node that routes execution based on order status.
- **Configuration choices:**
  - Condition type: string equals
  - Left value: `={{ $json.Status }}`
  - Right value: `Delivered`
  - Combinator: `and` with a single condition
- **Key expressions or variables used:**
  - `{{ $json.Status }}`
- **Input and output connections:**
  - Input from **Get an order**
  - True output to **Send a message to customer**
  - False output to **Send a message to the team**
- **Version-specific requirements:**
  - Uses type version `2.3`
  - Condition options set to strict type validation
- **Edge cases or potential failure types:**
  - `Status` field may not exist in Shopify response
  - Value may differ in capitalization or wording, such as `delivered`, `fulfilled`, or another logistics state
  - Branch may incorrectly route if data shape from Shopify differs from expectation
- **Sub-workflow reference:** None

**Important integration note:**  
The workflow assumes Shopify outputs a top-level field named `Status`. Depending on the actual Shopify node response, this may not be the right field. Some stores use fields like `fulfillment_status`, `financial_status`, or nested structures. This should be verified in execution data.

---

## Block 5: Notifications and Logging

### Overview

This block sends status-specific emails and records the result into Google Sheets. The true branch logs an auto-approved refund; the false branch notifies the team, updates the customer, and then logs the manual-review outcome.

### Nodes Involved

- Send a message to customer
- Send a message to the team
- Send a message to customer for "Pending" Status
- Logs to Sheet

### Node Details

#### 1. Send a message to customer

- **Type and technical role:** `n8n-nodes-base.gmail`  
  Gmail send node for the auto-approval branch.
- **Configuration choices:**
  - Recipient: `={{ $('Gmail Trigger').item.json.From }}`
  - Subject: `Refund Approved: Order #...`
  - Message body confirms refund approval and expected credit timing
- **Key expressions or variables used:**
  - `{{ $('Gmail Trigger').item.json.From }}`
  - `{{ $('Code in JavaScript').item.json.order_id }}`
- **Input and output connections:**
  - Input from **If** true branch
  - Output to **Logs to Sheet**
- **Version-specific requirements:**
  - Uses type version `2.2`
  - Requires Gmail OAuth2 send permission
- **Edge cases or potential failure types:**
  - `From` field may include a display name plus email address, which may or may not be accepted as-is
  - Gmail send quota or API permissions issues
  - Sending approval without actually creating a refund in Shopify could create business inconsistency
- **Sub-workflow reference:** None

#### 2. Send a message to the team

- **Type and technical role:** `n8n-nodes-base.gmail`  
  Internal escalation email for requests not auto-approved.
- **Configuration choices:**
  - Recipient: `<TEAM_EMAIL>` placeholder
  - Subject indicates manual review required
  - Message includes:
    - extracted order ID
    - refund reason
    - current status from Shopify
- **Key expressions or variables used:**
  - `{{ $('Code in JavaScript').item.json.order_id }}`
  - `{{ $('Code in JavaScript').item.json.reason }}`
  - `{{ $('Get row(s) in sheet').item.json.Status }}`
- **Input and output connections:**
  - Input from **If** false branch
  - Output to **Send a message to customer for "Pending" Status**
- **Version-specific requirements:**
  - Uses type version `2.2`
  - Requires Gmail OAuth2 send permission
- **Edge cases or potential failure types:**
  - `<TEAM_EMAIL>` is a placeholder and must be replaced before production use
  - Expression references `$('Get row(s) in sheet')`, but no such node exists in this workflow
  - This broken expression is likely to cause runtime failure or empty values
- **Sub-workflow reference:** None

**Critical issue:**  
This node references a non-existent node named **Get row(s) in sheet**. It should likely reference **Get an order** or another intended status source. As written, this is an implementation error.

#### 3. Send a message to customer for "Pending" Status

- **Type and technical role:** `n8n-nodes-base.gmail`  
  Customer-facing notification for the manual-review branch.
- **Configuration choices:**
  - Recipient: sender from Gmail trigger
  - Subject: update on refund request
  - Message says the request was received and requires manual review because of current order status
- **Key expressions or variables used:**
  - `{{ $('Gmail Trigger').item.json.From }}`
  - `{{ $('Get row(s) in sheet').item.json.Status }}`
  - `{{ $('Code in JavaScript').item.json.order_id }}`
- **Input and output connections:**
  - Input from **Send a message to the team**
  - Output to **Logs to Sheet**
- **Version-specific requirements:**
  - Uses type version `2.2`
  - Requires Gmail OAuth2 send permission
- **Edge cases or potential failure types:**
  - Same broken node reference to `Get row(s) in sheet`
  - Poor message formatting because there is no space after `as`
  - Recipient formatting concerns if `From` contains display-name syntax
- **Sub-workflow reference:** None

**Critical issue:**  
This node also references the missing node **Get row(s) in sheet**.

#### 4. Logs to Sheet

- **Type and technical role:** `n8n-nodes-base.googleSheets`  
  Appends a log entry to a Google Sheet for each processed request.
- **Configuration choices:**
  - Operation: `append`
  - Document: `Customer Orders`
  - Sheet/tab: `Refund_Logs`
  - Mapping mode: define manually
  - Columns written:
    - Date: `={{ $now }}`
    - Order ID: extracted order ID
    - Customer Email: Gmail sender
    - Reason: extracted reason
    - Outcome: auto-approved or pending manual review
    - Notes: original status text
- **Key expressions or variables used:**
  - `{{ $now }}`
  - `{{ $('Code in JavaScript').item.json.order_id }}`
  - `{{ $('Gmail Trigger').item.json.From }}`
  - `{{ $('Code in JavaScript').item.json.reason }}`
  - `{{ $('If').item.json.Status === 'Delivered' ? 'Auto-Approved' : 'Pending Manual Review ' }}`
  - `{{ $('Get row(s) in sheet').item.json.Status }}`
- **Input and output connections:**
  - Input from:
    - **Send a message to customer**
    - **Send a message to customer for "Pending" Status**
  - No downstream output
- **Version-specific requirements:**
  - Uses type version `4.7`
  - Requires Google Sheets OAuth2 credentials
  - Target sheet must exist and allow append access
- **Edge cases or potential failure types:**
  - Again references non-existent node `Get row(s) in sheet`
  - Outcome expression references `$('If').item.json.Status`; depending on branch behavior and item linking, this may not always be reliable
  - Sheet column names must exactly match configured schema
  - Permission errors or missing sheet tab
- **Sub-workflow reference:** None

**Critical issue:**  
This node also references **Get row(s) in sheet**, which does not exist. Logging notes will fail unless corrected.

---

## Non-Executable Documentation Nodes

These nodes are not part of runtime logic but provide important in-canvas documentation.

### Sticky Note

- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Purpose:** Global workflow description, setup requirements, and customization ideas.
- **Content summary:**
  - Describes the workflow as an automated refund handler
  - Notes required credentials:
    - Gmail OAuth2
    - Groq API key
    - Shopify access token with `read_orders`
    - Google Sheets OAuth2
  - Notes required sheet columns:
    - Date, Order ID, Customer Email, Reason, Outcome, Notes
  - Suggests enhancements like date filters and Slack/Discord alerts

### Sticky Note1

- Describes **Step 1: Intake & AI Extraction**

### Sticky Note2

- Describes **Step 2: Shopify Validation**

### Sticky Note3

- Describes **Step 3: Decision Logic**

### Sticky Note4

- Describes **Step 4: Notifications & Logging**

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Gmail Trigger | n8n-nodes-base.gmailTrigger | Poll Gmail for unread refund request emails |  | AI Agent2 | # Automated Refund Request Handler<br><br>### How it works<br>This workflow automates refund handling from incoming Gmail messages. It detects emails with “Refund Request”, extracts the Order ID and reason using AI, and fetches real-time order data from Shopify. Based on the order status, it decides whether the refund can be auto-approved or requires manual review. Customers are notified automatically, and every action is logged into Google Sheets for tracking and analytics.<br><br>### Setup steps<br>1. Connect Gmail OAuth2 (set to production in Google Cloud).<br>2. Add Groq API key for AI processing.<br>3. Connect Shopify using an access token with read_orders permission.<br>4. Connect Google Sheets OAuth2 for logging.<br>5. Ensure your sheet contains columns: Date, Order ID, Customer Email, Reason, Outcome, Notes.<br><br>### Customization tips<br>- Add a date filter to allow refunds only within a specific time window.<br>- Replace Gmail alerts with Slack or Discord for faster team response.<br>- Extend logic to check payment status before approving refunds.<br><br>## Step 1: Intake & AI Extraction<br>Triggers on refund emails.<br>Extracts order ID and reason using AI. |
| Groq Chat Model | @n8n/n8n-nodes-langchain.lmChatGroq | Provide LLM backend for extraction |  | AI Agent2 | # Automated Refund Request Handler<br><br>### How it works<br>This workflow automates refund handling from incoming Gmail messages. It detects emails with “Refund Request”, extracts the Order ID and reason using AI, and fetches real-time order data from Shopify. Based on the order status, it decides whether the refund can be auto-approved or requires manual review. Customers are notified automatically, and every action is logged into Google Sheets for tracking and analytics.<br><br>### Setup steps<br>1. Connect Gmail OAuth2 (set to production in Google Cloud).<br>2. Add Groq API key for AI processing.<br>3. Connect Shopify using an access token with read_orders permission.<br>4. Connect Google Sheets OAuth2 for logging.<br>5. Ensure your sheet contains columns: Date, Order ID, Customer Email, Reason, Outcome, Notes.<br><br>### Customization tips<br>- Add a date filter to allow refunds only within a specific time window.<br>- Replace Gmail alerts with Slack or Discord for faster team response.<br>- Extend logic to check payment status before approving refunds.<br><br>## Step 1: Intake & AI Extraction<br>Triggers on refund emails.<br>Extracts order ID and reason using AI. |
| AI Agent2 | @n8n/n8n-nodes-langchain.agent | Extract order ID and refund reason from email content | Gmail Trigger, Groq Chat Model | Code in JavaScript | # Automated Refund Request Handler<br><br>### How it works<br>This workflow automates refund handling from incoming Gmail messages. It detects emails with “Refund Request”, extracts the Order ID and reason using AI, and fetches real-time order data from Shopify. Based on the order status, it decides whether the refund can be auto-approved or requires manual review. Customers are notified automatically, and every action is logged into Google Sheets for tracking and analytics.<br><br>### Setup steps<br>1. Connect Gmail OAuth2 (set to production in Google Cloud).<br>2. Add Groq API key for AI processing.<br>3. Connect Shopify using an access token with read_orders permission.<br>4. Connect Google Sheets OAuth2 for logging.<br>5. Ensure your sheet contains columns: Date, Order ID, Customer Email, Reason, Outcome, Notes.<br><br>### Customization tips<br>- Add a date filter to allow refunds only within a specific time window.<br>- Replace Gmail alerts with Slack or Discord for faster team response.<br>- Extend logic to check payment status before approving refunds.<br><br>## Step 1: Intake & AI Extraction<br>Triggers on refund emails.<br>Extracts order ID and reason using AI. |
| Code in JavaScript | n8n-nodes-base.code | Parse AI JSON string into structured fields | AI Agent2 | Get an order | # Automated Refund Request Handler<br><br>### How it works<br>This workflow automates refund handling from incoming Gmail messages. It detects emails with “Refund Request”, extracts the Order ID and reason using AI, and fetches real-time order data from Shopify. Based on the order status, it decides whether the refund can be auto-approved or requires manual review. Customers are notified automatically, and every action is logged into Google Sheets for tracking and analytics.<br><br>### Setup steps<br>1. Connect Gmail OAuth2 (set to production in Google Cloud).<br>2. Add Groq API key for AI processing.<br>3. Connect Shopify using an access token with read_orders permission.<br>4. Connect Google Sheets OAuth2 for logging.<br>5. Ensure your sheet contains columns: Date, Order ID, Customer Email, Reason, Outcome, Notes.<br><br>### Customization tips<br>- Add a date filter to allow refunds only within a specific time window.<br>- Replace Gmail alerts with Slack or Discord for faster team response.<br>- Extend logic to check payment status before approving refunds.<br><br>## Step 1: Intake & AI Extraction<br>Triggers on refund emails.<br>Extracts order ID and reason using AI. |
| Get an order | n8n-nodes-base.shopify | Fetch Shopify order data by extracted order ID | Code in JavaScript | If | # Automated Refund Request Handler<br><br>### How it works<br>This workflow automates refund handling from incoming Gmail messages. It detects emails with “Refund Request”, extracts the Order ID and reason using AI, and fetches real-time order data from Shopify. Based on the order status, it decides whether the refund can be auto-approved or requires manual review. Customers are notified automatically, and every action is logged into Google Sheets for tracking and analytics.<br><br>### Setup steps<br>1. Connect Gmail OAuth2 (set to production in Google Cloud).<br>2. Add Groq API key for AI processing.<br>3. Connect Shopify using an access token with read_orders permission.<br>4. Connect Google Sheets OAuth2 for logging.<br>5. Ensure your sheet contains columns: Date, Order ID, Customer Email, Reason, Outcome, Notes.<br><br>### Customization tips<br>- Add a date filter to allow refunds only within a specific time window.<br>- Replace Gmail alerts with Slack or Discord for faster team response.<br>- Extend logic to check payment status before approving refunds.<br><br>## Step 2: Shopify Validation<br>Fetches order details from Shopify.<br>Used to verify refund eligibility. |
| If | n8n-nodes-base.if | Route auto-approval vs manual review based on status | Get an order | Send a message to customer, Send a message to the team | # Automated Refund Request Handler<br><br>### How it works<br>This workflow automates refund handling from incoming Gmail messages. It detects emails with “Refund Request”, extracts the Order ID and reason using AI, and fetches real-time order data from Shopify. Based on the order status, it decides whether the refund can be auto-approved or requires manual review. Customers are notified automatically, and every action is logged into Google Sheets for tracking and analytics.<br><br>### Setup steps<br>1. Connect Gmail OAuth2 (set to production in Google Cloud).<br>2. Add Groq API key for AI processing.<br>3. Connect Shopify using an access token with read_orders permission.<br>4. Connect Google Sheets OAuth2 for logging.<br>5. Ensure your sheet contains columns: Date, Order ID, Customer Email, Reason, Outcome, Notes.<br><br>### Customization tips<br>- Add a date filter to allow refunds only within a specific time window.<br>- Replace Gmail alerts with Slack or Discord for faster team response.<br>- Extend logic to check payment status before approving refunds.<br><br>## Step 3: Decision Logic<br>Checks if order is delivered.<br>Routes to auto-approve or manual review. |
| Send a message to customer | n8n-nodes-base.gmail | Send auto-approval email to customer | If | Logs to Sheet | # Automated Refund Request Handler<br><br>### How it works<br>This workflow automates refund handling from incoming Gmail messages. It detects emails with “Refund Request”, extracts the Order ID and reason using AI, and fetches real-time order data from Shopify. Based on the order status, it decides whether the refund can be auto-approved or requires manual review. Customers are notified automatically, and every action is logged into Google Sheets for tracking and analytics.<br><br>### Setup steps<br>1. Connect Gmail OAuth2 (set to production in Google Cloud).<br>2. Add Groq API key for AI processing.<br>3. Connect Shopify using an access token with read_orders permission.<br>4. Connect Google Sheets OAuth2 for logging.<br>5. Ensure your sheet contains columns: Date, Order ID, Customer Email, Reason, Outcome, Notes.<br><br>### Customization tips<br>- Add a date filter to allow refunds only within a specific time window.<br>- Replace Gmail alerts with Slack or Discord for faster team response.<br>- Extend logic to check payment status before approving refunds.<br><br>## Step 4: Notifications & Logging<br>Sends customer/team emails.<br>Logs outcome into Google Sheets. |
| Send a message to the team | n8n-nodes-base.gmail | Alert internal team for manual review | If | Send a message to customer for "Pending" Status | # Automated Refund Request Handler<br><br>### How it works<br>This workflow automates refund handling from incoming Gmail messages. It detects emails with “Refund Request”, extracts the Order ID and reason using AI, and fetches real-time order data from Shopify. Based on the order status, it decides whether the refund can be auto-approved or requires manual review. Customers are notified automatically, and every action is logged into Google Sheets for tracking and analytics.<br><br>### Setup steps<br>1. Connect Gmail OAuth2 (set to production in Google Cloud).<br>2. Add Groq API key for AI processing.<br>3. Connect Shopify using an access token with read_orders permission.<br>4. Connect Google Sheets OAuth2 for logging.<br>5. Ensure your sheet contains columns: Date, Order ID, Customer Email, Reason, Outcome, Notes.<br><br>### Customization tips<br>- Add a date filter to allow refunds only within a specific time window.<br>- Replace Gmail alerts with Slack or Discord for faster team response.<br>- Extend logic to check payment status before approving refunds.<br><br>## Step 4: Notifications & Logging<br>Sends customer/team emails.<br>Logs outcome into Google Sheets. |
| Send a message to customer for "Pending" Status | n8n-nodes-base.gmail | Inform customer that manual review is required | Send a message to the team | Logs to Sheet | # Automated Refund Request Handler<br><br>### How it works<br>This workflow automates refund handling from incoming Gmail messages. It detects emails with “Refund Request”, extracts the Order ID and reason using AI, and fetches real-time order data from Shopify. Based on the order status, it decides whether the refund can be auto-approved or requires manual review. Customers are notified automatically, and every action is logged into Google Sheets for tracking and analytics.<br><br>### Setup steps<br>1. Connect Gmail OAuth2 (set to production in Google Cloud).<br>2. Add Groq API key for AI processing.<br>3. Connect Shopify using an access token with read_orders permission.<br>4. Connect Google Sheets OAuth2 for logging.<br>5. Ensure your sheet contains columns: Date, Order ID, Customer Email, Reason, Outcome, Notes.<br><br>### Customization tips<br>- Add a date filter to allow refunds only within a specific time window.<br>- Replace Gmail alerts with Slack or Discord for faster team response.<br>- Extend logic to check payment status before approving refunds.<br><br>## Step 4: Notifications & Logging<br>Sends customer/team emails.<br>Logs outcome into Google Sheets. |
| Logs to Sheet | n8n-nodes-base.googleSheets | Append audit log row to Google Sheets | Send a message to customer, Send a message to customer for "Pending" Status |  | # Automated Refund Request Handler<br><br>### How it works<br>This workflow automates refund handling from incoming Gmail messages. It detects emails with “Refund Request”, extracts the Order ID and reason using AI, and fetches real-time order data from Shopify. Based on the order status, it decides whether the refund can be auto-approved or requires manual review. Customers are notified automatically, and every action is logged into Google Sheets for tracking and analytics.<br><br>### Setup steps<br>1. Connect Gmail OAuth2 (set to production in Google Cloud).<br>2. Add Groq API key for AI processing.<br>3. Connect Shopify using an access token with read_orders permission.<br>4. Connect Google Sheets OAuth2 for logging.<br>5. Ensure your sheet contains columns: Date, Order ID, Customer Email, Reason, Outcome, Notes.<br><br>### Customization tips<br>- Add a date filter to allow refunds only within a specific time window.<br>- Replace Gmail alerts with Slack or Discord for faster team response.<br>- Extend logic to check payment status before approving refunds.<br><br>## Step 4: Notifications & Logging<br>Sends customer/team emails.<br>Logs outcome into Google Sheets. |
| Sticky Note | n8n-nodes-base.stickyNote | Canvas documentation |  |  |  |
| Sticky Note1 | n8n-nodes-base.stickyNote | Canvas documentation for intake and AI step |  |  |  |
| Sticky Note2 | n8n-nodes-base.stickyNote | Canvas documentation for Shopify validation |  |  |  |
| Sticky Note3 | n8n-nodes-base.stickyNote | Canvas documentation for decision step |  |  |  |
| Sticky Note4 | n8n-nodes-base.stickyNote | Canvas documentation for notifications and logging |  |  |  |

---

# 4. Reproducing the Workflow from Scratch

Below is a full rebuild sequence for n8n.

## Prerequisites

1. **Gmail OAuth2 credentials**
   - One credential for the Gmail Trigger
   - Same or another Gmail credential for sending emails
   - In Google Cloud, ensure the OAuth consent and Gmail scopes are configured correctly
   - Production publishing is recommended for stable use

2. **Groq API credentials**
   - Add your Groq API key in n8n

3. **Shopify credentials**
   - Use an access token
   - Ensure the app/token has at least `read_orders` permission

4. **Google Sheets OAuth2 credentials**
   - The target spreadsheet must be accessible by the authenticated Google account

5. **Target Google Sheet**
   - Spreadsheet name: `Customer Orders`
   - Worksheet/tab name: `Refund_Logs`
   - Required columns:
     - Date
     - Order ID
     - Customer Email
     - Reason
     - Outcome
     - Notes

---

## Build Steps

1. **Create a new workflow**  
   Name it: **Process refund requests from Gmail using Shopify and Google Sheets**

2. **Add a Gmail Trigger node**
   - Node type: **Gmail Trigger**
   - Polling frequency: **Every Minute**
   - Filter query: `subject:"Refund Request"`
   - Read status: **Unread**
   - Connect Gmail OAuth2 credentials

3. **Add a Groq Chat Model node**
   - Node type: **Groq Chat Model**
   - Model: `llama-3.3-70b-versatile`
   - Connect Groq API credentials

4. **Add an AI Agent node**
   - Node type: **AI Agent**
   - Set prompt mode to define the prompt manually
   - Use this prompt content:

     ```text
     Extract the Order ID and the Reason for the refund from the email details provided below. 

     Instructions:
     - Look for the Order ID (usually a 4-digit number like 1001).
     - Summarize the reason for the refund in 5 words or less.
     - Respond ONLY with a clean JSON object. No extra text, no "Here is your data", just JSON.

     Desired Format:
     {
       "order_id": "value",
       "reason": "value"
     }

     Email Data:
     Subject: {{ $json.Subject }}
     Snippet: {{ $json.snippet }}
     ```

5. **Connect the model to the AI Agent**
   - Connect **Groq Chat Model** to **AI Agent** using the AI language model connection
   - Connect **Gmail Trigger** to **AI Agent** using the main connection

6. **Add a Code node**
   - Node type: **Code**
   - Language: JavaScript
   - Paste this logic:

     ```javascript
     // Get the output string from the AI Agent
     const aiRawOutput = items[0].json.output;

     // Parse the string into a real JSON object
     const cleanData = JSON.parse(aiRawOutput);

     // Return the data in a way n8n understands
     return [
       {
         json: {
           order_id: cleanData.order_id,
           reason: cleanData.reason,
           original_summary: cleanData.reason
         }
       }
     ];
     ```

   - Connect **AI Agent** to **Code**

7. **Add a Shopify node**
   - Node type: **Shopify**
   - Operation: **Get Order**
   - Authentication: **Access Token**
   - Order ID field: `{{ $json.order_id }}`
   - Connect Shopify credentials
   - Connect **Code** to **Shopify**

8. **Validate the Shopify lookup format**
   - Before going live, test whether your extracted order ID matches what Shopify expects
   - If Shopify expects internal order IDs instead of visible order numbers, replace this step with an order search approach

9. **Add an If node**
   - Node type: **If**
   - Condition:
     - Left value: `{{ $json.Status }}`
     - Operator: **Equals**
     - Right value: `Delivered`
   - Connect **Shopify** to **If**

10. **Add the auto-approval Gmail node**
    - Node type: **Gmail**
    - Operation: **Send**
    - To: `{{ $('Gmail Trigger').item.json.From }}`
    - Subject:

      ```text
      Refund Approved: Order #{{ $('Code in JavaScript').item.json.order_id }}
      ```

    - Message:

      ```text
      Hi there, Good news! We have reviewed your request for Order #{{ $('Code in JavaScript').item.json.order_id }} and your refund has been approved. You should see the credit in your account within 5-7 business days.
      ```

    - Connect this node to the **true** output of the If node
    - Connect Gmail sending credentials

11. **Add the team notification Gmail node**
    - Node type: **Gmail**
    - Operation: **Send**
    - To: replace `<TEAM_EMAIL>` with a real internal email address
    - Subject:

      ```text
      ACTION REQUIRED: Manual Refund Review (Order {{ $('Code in JavaScript').item.json.order_id }})
      ```

    - Recommended corrected message body:

      ```text
      A refund request was received but could not be auto-approved.

      Order ID: {{ $('Code in JavaScript').item.json.order_id }}
      Reason: {{ $('Code in JavaScript').item.json.reason }}
      Current Status: {{ $('Get an order').item.json.Status }}
      ```

    - Connect this node to the **false** output of the If node

12. **Add the pending-status customer Gmail node**
    - Node type: **Gmail**
    - Operation: **Send**
    - To: `{{ $('Gmail Trigger').item.json.From }}`
    - Subject:

      ```text
      Update on your Refund Request: Order #{{ $('Code in JavaScript').item.json.order_id }}
      ```

    - Recommended corrected message body:

      ```text
      Hi, we've received your refund request. Because your order is currently marked as {{ $('Get an order').item.json.Status }}, a member of our team needs to review this manually. We will get back to you shortly.
      ```

    - Connect **Send a message to the team** to this node

13. **Add a Google Sheets node**
    - Node type: **Google Sheets**
    - Operation: **Append**
    - Spreadsheet/document: `Customer Orders`
    - Sheet/tab: `Refund_Logs`
    - Mapping mode: define fields manually
    - Map columns as follows:

      - **Date** → `{{ $now }}`
      - **Order ID** → `{{ $('Code in JavaScript').item.json.order_id }}`
      - **Customer Email** → `{{ $('Gmail Trigger').item.json.From }}`
      - **Reason** → `{{ $('Code in JavaScript').item.json.reason }}`
      - **Outcome** → use a branch-safe value, for example:
        - On auto-approved branch: set a static or expression value of `Auto-Approved`
        - On manual-review branch: set `Pending Manual Review`
      - **Notes** → recommended:
        `Original Status: {{ $('Get an order').item.json.Status }}`

14. **Connect logging**
    - Connect **Send a message to customer** to **Logs to Sheet**
    - Connect **Send a message to customer for "Pending" Status** to **Logs to Sheet**

15. **Correct the broken expressions from the original workflow**
    - Replace all references to:
      - `$('Get row(s) in sheet').item.json.Status`
    - With:
      - `$('Get an order').item.json.Status`
    - This correction is required in:
      - team email
      - pending customer email
      - Google Sheets notes field

16. **Optionally improve branch-specific logging**
    - The original workflow uses:
      `{{ $('If').item.json.Status === 'Delivered' ? 'Auto-Approved' : 'Pending Manual Review ' }}`
    - A more reliable design is to use separate Set nodes before logging or separate Logs nodes per branch
    - This avoids branch ambiguity

17. **Add sticky notes if desired**
    - Add a global note describing the workflow
    - Add section notes for:
      - Intake & AI Extraction
      - Shopify Validation
      - Decision Logic
      - Notifications & Logging

18. **Test each stage**
    - Send a sample unread Gmail with subject `Refund Request`
    - Include a visible order number and refund reason in the message
    - Confirm:
      - AI returns valid JSON
      - Code node parses successfully
      - Shopify lookup works
      - If node receives a valid `Status`
      - Correct branch emails are sent
      - Sheet row is appended

19. **Activate the workflow**
    - Once credentials, expressions, and test data are validated, activate the workflow

---

## Recommended Hardening Changes

To make the workflow production-safe, implement these improvements:

1. **Handle invalid AI output**
   - Add a Try/Catch strategy or an If node after parsing
   - Route malformed JSON to manual review instead of failing the workflow

2. **Normalize customer email**
   - Parse the raw Gmail `From` field if needed
   - Some messages may include `Name <email@example.com>`

3. **Verify actual Shopify status field**
   - Inspect the Shopify response and confirm whether `Status` is correct
   - Adjust to the real field name if needed

4. **Distinguish visible order number from Shopify internal ID**
   - This is one of the most likely integration issues

5. **Avoid approving refunds by email only**
   - Current logic sends an approval email but does not actually create a refund transaction in Shopify

6. **Prevent duplicate processing**
   - Consider marking processed Gmail messages as read or labeling them after success

7. **Add timeout/error notifications**
   - Notify the internal team if Gmail, Groq, Shopify, or Google Sheets fails

---

## Explicit Entry Points and Sub-Workflow Notes

- **Entry point:** `Gmail Trigger`
- **Additional entry points:** None
- **Sub-workflows invoked:** None
- **Sub-workflow nodes present:** None

---

## Key Implementation Issues in the Provided Workflow

Before using this workflow as-is, correct these items:

1. **Broken node references**
   - `Get row(s) in sheet` is referenced multiple times but does not exist

2. **Potential Shopify ID mismatch**
   - Extracted order ID may not match Shopify’s expected order identifier

3. **Assumed status field**
   - `Status` may not be the actual field returned by Shopify

4. **Approval without refund execution**
   - The workflow notifies the customer but does not create a refund in Shopify

These issues do not change the overall design intent, but they do affect runtime reliability and business correctness.