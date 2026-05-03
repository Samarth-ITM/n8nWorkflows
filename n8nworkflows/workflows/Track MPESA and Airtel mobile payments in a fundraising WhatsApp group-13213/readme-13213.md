Track MPESA and Airtel mobile payments in a fundraising WhatsApp group

https://n8nworkflows.xyz/workflows/track-mpesa-and-airtel-mobile-payments-in-a-fundraising-whatsapp-group-13213


# Track MPESA and Airtel mobile payments in a fundraising WhatsApp group

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Track MPESA and Airtel mobile payments in a fundraising WhatsApp group  
**Workflow name (in JSON):** Mobile Payments

**Purpose:**  
This workflow receives incoming WhatsApp messages (via Twilio → n8n Webhook), detects whether each message is an M-PESA receipt, an Airtel Money receipt, a “summary/total” request, or a “close campaign” command, then logs payments to an n8n Data Table, generates a running summary, and replies back into WhatsApp with the updated summary (or a closure confirmation).

**Typical use cases:**
- Treasurers tracking contributions in a WhatsApp fundraising group.
- Maintaining a running total without manual tallying.
- Resetting/closing a campaign by deleting recorded payment rows.

**Logical blocks (by dependency and function):**
1.1 **Inbound WhatsApp Reception & Normalization** (Webhook → field shaping)  
1.2 **Message Classification & Routing** (regex-based classification + Switch routes)  
1.3 **Payment Parsing & Persistence** (M-PESA/Airtel parsing → save to Data Table)  
1.4 **Summary / Clean Campaign Operations** (fetch rows → summarize, or delete rows → close message)  
1.5 **Reply Assembly & WhatsApp Response** (merge metadata + generated message → validate → Twilio send)

---

## 2. Block-by-Block Analysis

### 2.1 Inbound WhatsApp Reception & Normalization

**Overview:**  
Receives a POST webhook (intended for Twilio WhatsApp inbound), extracts `From`, `To`, and `Body` into normalized fields used downstream.

**Nodes involved:**
- **Webhook**
- **Edit Fields**

#### Node: Webhook
- **Type / role:** `n8n-nodes-base.webhook` — entry point receiving Twilio inbound HTTP requests.
- **Key configuration:**
  - **HTTP Method:** POST
  - **Path:** `mobile-payments`
  - **Response mode:** “responseNode” (workflow is expected to respond via a dedicated Response node, but this workflow actually responds via Twilio outbound instead; no Response node is present).
- **Inputs/Outputs:**
  - **Output →** Edit Fields
- **Edge cases / failures:**
  - If Twilio is configured for URL-encoded form payloads, ensure n8n parses `body` as expected. This workflow expects `{$json.body.From, $json.body.To, $json.body.Body}`.
  - Because `responseMode` is set to `responseNode` but no response node exists, the webhook request may not get an HTTP response in the way n8n expects. Twilio generally tolerates quick empty/200 responses; in some setups, this mismatch can cause retries/timeouts.
- **Version notes:** typeVersion 1.

#### Node: Edit Fields
- **Type / role:** `n8n-nodes-base.set` — creates normalized fields for downstream logic.
- **Configuration choices (interpreted):**
  - Sets:
    - `from` = `{{$json.body.From}}`
    - `to` = `{{$json.body.To}}`
    - `message` = `{{$json.body.Body}}`
- **Inputs/Outputs:**
  - **Input ←** Webhook
  - **Output →** Classify Message
- **Edge cases / failures:**
  - If Twilio sends different casing/keys (e.g., `Body` missing), `message` may become `undefined`, affecting classification.

---

### 2.2 Message Classification & Routing

**Overview:**  
Classifies inbound text into one of: `mpesa`, `airtel`, `summary`, `clean`, or `unknown` using simple keyword checks and regex checks, then routes to the appropriate branch.

**Nodes involved:**
- **Classify Message**
- **Route**
- **Drop original message** (runs in parallel to preserve metadata for reply)

#### Node: Classify Message
- **Type / role:** `n8n-nodes-base.code` — custom JS classifier.
- **Configuration choices:**
  - Produces `{ paymentType, message, from, to }`.
  - Keyword routing:
    - `summary` if message contains any of: `summary`, `total`, `progress`
    - `clean` if message contains any of: `close`, `restart`, `refresh`, `delete`, `end`
  - Regex routing:
    - `airtel` if message starts with `TID:...`
    - `mpesa` if message starts with `<CODE> Confirmed`
  - Otherwise `unknown`.
- **Key variables:**
  - `message = ($json.message || '').trim()`
  - `text = message.toLowerCase()`
- **Inputs/Outputs:**
  - **Input ←** Edit Fields
  - **Output (main) →**
    - Route
    - Drop original message (parallel path)
- **Edge cases / failures:**
  - False positives: any chat message containing “total” could trigger a summary.
  - Locale/format variance in SMS receipts may cause misclassification (e.g., different wording).
- **Version notes:** typeVersion 2.

#### Node: Route
- **Type / role:** `n8n-nodes-base.switch` — routes based on `paymentType`.
- **Configuration choices:**
  - Creates named outputs: `unknown`, `clean`, `summary`, `mpesa`, `airtel`
  - Compares `{{$json.paymentType}}` equals each literal.
- **Inputs/Outputs:**
  - **Input ←** Classify Message
  - **Outputs →**
    - `clean` → Delete payments
    - `summary` → Fetch payments
    - `mpesa` → Mpesa
    - `airtel` → Airtel Money
    - `unknown` → (no connection; message will not be replied to)
- **Edge cases / failures:**
  - `unknown` branch is unhandled; senders get no response for unsupported messages unless you add a fallback.
- **Version notes:** typeVersion 3.4.

#### Node: Drop original message
- **Type / role:** `n8n-nodes-base.set` — strips message content and keeps only metadata to be merged later.
- **Configuration choices:**
  - Sets:
    - `from = {{$json.from}}`
    - `to = {{$json.to}}`
- **Inputs/Outputs:**
  - **Input ←** Classify Message
  - **Output →** Merge (input 0)
- **Edge cases / failures:**
  - None major; ensures reply pipeline doesn’t accidentally reuse the inbound `message` text as outbound reply.
- **Version notes:** typeVersion 3.4.

---

### 2.3 Payment Parsing & Persistence

**Overview:**  
Extracts structured payment data from provider-specific receipt formats using regex, then writes the normalized payment record to an n8n Data Table.

**Nodes involved:**
- **Mpesa**
- **Airtel Money**
- **Save payment**

#### Node: Mpesa
- **Type / role:** `n8n-nodes-base.code` — parses M-PESA receipt format.
- **Configuration choices:**
  - Regex anchored at start:
    - `^([A-Z0-9]+)\s+Confirmed...received\s+Ksh([\d,.]+)\s+from\s+(.+?)\s+(\d{9,12})\s+on\s+(\d{1,2}\/\d{1,2}\/\d{2})\s+at\s+([\d:]+\s+(?:AM|PM))`
  - Output fields:
    - `provider: 'mpesa'`
    - `reference`, `amount` (Number + comma removal), `payer`, `phone`
    - `groupId: $json.from` (uses WhatsApp sender/chat id for grouping)
    - `date`, `time`
  - If no match: returns `{}` (an item with no `json` wrapper in this node’s code path is not emitted; effectively yields empty output for that item).
- **Inputs/Outputs:**
  - **Input ←** Route (mpesa)
  - **Output →** Save payment
- **Edge cases / failures:**
  - SMS wording variations (“Ksh 1.00”, spacing, “KES”) may fail parsing.
  - Phone number length constraints (9–12 digits) may exclude some formats.
- **Version notes:** typeVersion 2.

#### Node: Airtel Money
- **Type / role:** `n8n-nodes-base.code` — parses Airtel Money receipt format.
- **Configuration choices:**
  - Regex:
    - `/TID:([A-Z0-9]+).*?Received\s+Ksh\s*([\d,.]+)\s+from\s+(.+?)\s+(\d{9,15})\s+on\s+(\d{2}\/\d{2}\/\d{2})\s+([\d:]+\s+(?:AM|PM))/i`
  - Output fields:
    - `provider: 'airtel'`
    - `reference`, `amount`, `payer`, `phone`
    - `groupId: $json.from`
    - `date`, `time`
  - If no match: returns `{}`.
- **Inputs/Outputs:**
  - **Input ←** Route (airtel)
  - **Output →** Save payment
- **Edge cases / failures:**
  - If Airtel receipt format differs (currency label, date format, “hrs”), parsing fails silently.
  - Returns empty output on mismatch, so no logging and no reply.
- **Version notes:** typeVersion 2.

#### Node: Save payment
- **Type / role:** `n8n-nodes-base.dataTable` — inserts payment row into Data Table.
- **Configuration choices:**
  - **Operation:** insert (implicit by node + column mapping; configured schema with “auto map input data”)
  - **Data Table:** `mobile_payments` (ID in workflow: `QWbf1qYJeu6rTyyZ`)
  - **Columns defined:** `reference` (string), `payer` (string), `phone` (string), `amount` (number), `date` (string), `time` (string)
  - **Mapping mode:** Auto-map from input JSON by matching field names.
- **Inputs/Outputs:**
  - **Input ←** Mpesa OR Airtel Money
  - **Output →** Fetch payments
- **Edge cases / failures:**
  - If required columns exist in the table but missing in input, insert may fail.
  - If `amount` is not numeric (parsing failed), insert may fail or store null depending on table behavior.
- **Version notes:** typeVersion 1.1.

---

### 2.4 Summary / Clean Campaign Operations

**Overview:**  
Either fetches all payments for the conversation/group and generates a summary, or deletes all related payment rows and generates a closure message.

**Nodes involved:**
- **Fetch payments**
- **Create Summary**
- **Delete payments**
- **Create Message**

#### Node: Fetch payments
- **Type / role:** `n8n-nodes-base.dataTable` — queries logged payments.
- **Configuration choices:**
  - **Operation:** get
  - **Filters:** two conditions on `groupId`:
    - `groupId = {{$json.from}}`
    - `groupId = {{$json.groupId}}`
  - The intent appears to be: fetch by group id or sender; however, with typical “AND” semantics this would require both to match (and one of them may be undefined), which can yield empty results.
- **Inputs/Outputs:**
  - **Input ←** Route (summary) OR Save payment (after insert)
  - **Output →** Create Summary
- **Edge cases / failures:**
  - **High risk logic issue:** dual `groupId` filter conditions can prevent matches if `$json.groupId` is not set (it is set in Mpesa/Airtel parsing outputs, but not in summary branch where `groupId` isn’t created—only `from` exists).
  - If you want “OR” behavior, you must restructure filtering (or ensure one consistent field is always present).
- **Version notes:** typeVersion 1.1.

#### Node: Create Summary
- **Type / role:** `n8n-nodes-base.function` — formats payment list and total.
- **Configuration choices:**
  - If no `items`: returns `"No payments recorded yet."`
  - Sums `row.amount` when it is a number
  - Builds lines: `"{index}. {payer} – Ksh {amount}"`
  - Message includes a header and total; note it contains a “💰” symbol.
- **Inputs/Outputs:**
  - **Input ←** Fetch payments
  - **Output →** Merge (input 1)
- **Edge cases / failures:**
  - If Data Table returns amounts as strings, the defensive check (`typeof row.amount !== 'number'`) will skip them, producing total = 0 and empty list.
  - WhatsApp formatting: the total uses asterisks for bold; that’s fine for WhatsApp.
- **Version notes:** typeVersion 1.

#### Node: Delete payments
- **Type / role:** `n8n-nodes-base.dataTable` — deletes payment rows to “close/reset” a campaign.
- **Configuration choices:**
  - **Operation:** deleteRows
  - **Filters:** same dual `groupId` conditions as Fetch payments:
    - `groupId = {{$json.from}}`
    - `groupId = {{$json.groupId}}`
- **Inputs/Outputs:**
  - **Input ←** Route (clean)
  - **Output →** Create Message
- **Edge cases / failures:**
  - Same **AND vs OR** risk as Fetch payments; may fail to delete anything if one value is undefined.
- **Version notes:** typeVersion 1.1.

#### Node: Create Message
- **Type / role:** `n8n-nodes-base.function` — creates a fixed closure confirmation message.
- **Configuration choices:**
  - Outputs:
    - `message: "Payment collection campaign has been closed. Details deleted"`
- **Inputs/Outputs:**
  - **Input ←** Delete payments
  - **Output →** Merge (input 1)
- **Edge cases / failures:** none significant.
- **Version notes:** typeVersion 1.

---

### 2.5 Reply Assembly & WhatsApp Response

**Overview:**  
Combines metadata (`from`, `to`) with the generated message, validates it, then sends it via Twilio WhatsApp outbound.

**Nodes involved:**
- **Merge**
- **Build one object**
- **Valid message ?**
- **Send a WhatsApp message**

#### Node: Merge
- **Type / role:** `n8n-nodes-base.merge` — synchronizes two streams: metadata + generated message.
- **Configuration choices:**
  - Uses default Merge behavior (not explicitly set). Practically, it is being used as a two-input join point:
    - Input 0: Drop original message (from/to)
    - Input 1: Create Summary OR Create Message (message)
- **Inputs/Outputs:**
  - **Input 0 ←** Drop original message
  - **Input 1 ←** Create Summary OR Create Message
  - **Output →** Build one object
- **Edge cases / failures:**
  - Merge defaults can be sensitive: if one branch produces multiple items and the other produces one, pairing may be unexpected.
- **Version notes:** typeVersion 3.2.

#### Node: Build one object
- **Type / role:** `n8n-nodes-base.code` — flattens/merges multiple input items into a single JSON object.
- **Configuration choices:**
  - Reduces all incoming items into one object: `{...item1.json, ...item2.json, ...}`
  - Returns a single item.
- **Inputs/Outputs:**
  - **Input ←** Merge
  - **Output →** Valid message ?
- **Edge cases / failures:**
  - Key collisions: later items overwrite earlier keys with the same name.
- **Version notes:** typeVersion 2.

#### Node: Valid message ?
- **Type / role:** `n8n-nodes-base.if` — ensures a reply message exists and is not empty.
- **Configuration choices:**
  - Conditions:
    - `{{$json.message}}` exists
    - `{{$json.message}}` not empty
  - True branch goes to Twilio send; false branch does nothing.
- **Inputs/Outputs:**
  - **Input ←** Build one object
  - **True →** Send a WhatsApp message
  - **False →** (unconnected)
- **Edge cases / failures:**
  - If upstream produced no `message` (e.g., parsing failed, unknown branch), nothing is sent.
- **Version notes:** typeVersion 2.3.

#### Node: Send a WhatsApp message
- **Type / role:** `n8n-nodes-base.twilio` — sends WhatsApp message via Twilio.
- **Configuration choices:**
  - **To:** `{{$json.from.replace('whatsapp:', '')}}`
  - **From:** `{{$json.to.replace('whatsapp:', '')}}`
  - **Message body:** `{{$json.message}}`
  - **toWhatsapp:** true
  - Uses Twilio credentials: “Twilio account”
- **Inputs/Outputs:**
  - **Input ←** Valid message ? (true path)
  - **Output:** none
- **Edge cases / failures:**
  - If `from/to` don’t start with `whatsapp:`, `.replace('whatsapp:', '')` is harmless but may result in invalid numbers if formatting is unexpected.
  - Twilio errors: sandbox limitations, WhatsApp template requirements (for outbound to users not in a session), auth errors, rate limits.
- **Version notes:** typeVersion 1.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Webhook | Webhook | Receive inbound Twilio WhatsApp webhook | — | Edit Fields | ## Try It Out!  \n### This n8n templates assists with keeping track of mobile payments within a fundraising WhatsApp group.  \n\nUse cases: We fundraise alot using whatsapp groups in East Africa, especially in Kenya ! Keeping track of each payment and the tallying requires alot of manual effort and brings unnecessary tension in cases of Errors of Commision or Ommision.  \n\nWorks with MPESA and AIRTEL MONEY, for now.  \n\n\n### How it works  \n* Connect you twillio account / mobile number to the webhook.  \n* First node keeps track of the from, to and message.  \n* We use simple regex to classify the text of the message.  \n* A switch node routes payment messages based on the *payment service provider*. The message may be a request for the current total or an instruction to end the campaign and clear the payment logs.  \n* Clearing may be necessary in case of mistakes since we provide no edit function. It may also be necessary to avoid mixing payments of a previous fundraising campaign with the current one.  \n* Payment information is extracted from the message accoding to the SMS format of the service provider. This is then saved to a data table.  \n* After each payment, or a request for summary, all payments related to the sender/groupid are fetched and taken to the next node for summarization.  \n* A merge node is used to bring in the message metadata (from, to, ) to assist in whatsapp reply via twillio  \n\n### How to use  \n* As the treasurer / payee keep SMS receipts of all incoming mobile payments.  \n* Each SMS receipt will contain the amount and the senders details, among other info.  \n* Send each one by one to the Twilio number via whatsapp alternatively add the twilio number to the whatssapp group.  \n\n### Requirements  \n* Twillio account  \n* Accessible webhook url  \n* This example uses a data table `payment_table` but an SQL node is recommended for production use  \n\n\n### Need Help?  \nJoin the [Discord](https://discord.com/invite/XPKeKXeB7d) or ask in the [Forum](https://community.n8n.io/)!  \n\n\\n\\nHappy Hacking! |
| Edit Fields | Set | Normalize Twilio fields into from/to/message | Webhook | Classify Message | (same as above) |
| Classify Message | Code | Detect message type (mpesa/airtel/summary/clean/unknown) | Edit Fields | Route; Drop original message | (same as above) |
| Route | Switch | Branch execution by paymentType | Classify Message | Delete payments; Fetch payments; Mpesa; Airtel Money | (same as above) |
| Mpesa | Code | Parse M-PESA receipt into structured payment fields | Route | Save payment | ## Log Payment  \nLog each payment posted in the group chat. |
| Airtel Money | Code | Parse Airtel Money receipt into structured payment fields | Route | Save payment | ## Log Payment  \nLog each payment posted in the group chat. |
| Save payment | Data Table | Insert parsed payment record | Mpesa; Airtel Money | Fetch payments | ## Log Payment  \nLog each payment posted in the group chat. |
| Fetch payments | Data Table | Retrieve all payments for a group/sender | Save payment; Route (summary) | Create Summary | ## Summarize payments  \nUsing group id or sender |
| Create Summary | Function | Format payment list + compute total | Fetch payments | Merge | ## Summarize payments  \nUsing group id or sender |
| Delete payments | Data Table | Delete all payments for a group/sender (close/reset) | Route (clean) | Create Message | (same as above) |
| Create Message | Function | Create fixed “campaign closed” reply | Delete payments | Merge | (same as above) |
| Drop original message | Set | Keep only from/to for replying | Classify Message | Merge | (same as above) |
| Merge | Merge | Join metadata with generated reply message | Drop original message; Create Summary/Create Message | Build one object |  |
| Build one object | Code | Flatten merged items into one JSON object | Merge | Valid message ? |  |
| Valid message ? | IF | Prevent sending empty messages | Build one object | Send a WhatsApp message (true) |  |
| Send a WhatsApp message | Twilio | Send WhatsApp reply via Twilio | Valid message ? | — |  |
| Sticky Note | Sticky Note | Comment block | — | — | ## Log Payment  \nLog each payment posted in the group chat. |
| Sticky Note1 | Sticky Note | Comment block | — | — | ## Summarize payments  \nUsing group id or sender |
| Sticky Note2 | Sticky Note | Empty comment container | — | — |  |
| Sticky Note3 | Sticky Note | Full workflow usage notes | — | — | ## Try It Out!  \n(see full content above; includes Discord/Forum links) |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow** in n8n named **“Mobile Payments”**.

2) **Add Webhook node**
   - Type: **Webhook**
   - Method: **POST**
   - Path: `mobile-payments`
   - Response Mode: set to **“Using ‘Respond to Webhook’ node”** *or* switch to “On Received” (recommended if you’re not adding a Response node).
   - Copy the **Test/Production URL** for Twilio configuration.

3) **Configure Twilio WhatsApp inbound**
   - In Twilio (WhatsApp sandbox or approved number), set **“When a message comes in”** to the Webhook URL.
   - Ensure Twilio sends parameters `From`, `To`, `Body` (standard Twilio inbound).

4) **Add Set node: “Edit Fields”**
   - Add 3 fields:
     - `from` (String) = `{{$json.body.From}}`
     - `to` (String) = `{{$json.body.To}}`
     - `message` (String) = `{{$json.body.Body}}`
   - Connect: **Webhook → Edit Fields**

5) **Add Code node: “Classify Message”**
   - Paste logic that:
     - reads `message/from/to`
     - assigns `paymentType` based on keywords + regex (mpesa/airtel/summary/clean/unknown)
     - outputs `{ paymentType, message, from, to }`
   - Connect: **Edit Fields → Classify Message**

6) **Add Switch node: “Route”**
   - Switch on `{{$json.paymentType}}`
   - Add outputs (equals):
     - `unknown`
     - `clean`
     - `summary`
     - `mpesa`
     - `airtel`
   - Connect: **Classify Message → Route**

7) **Add Set node: “Drop original message”**
   - Keep only:
     - `from` = `{{$json.from}}`
     - `to` = `{{$json.to}}`
   - Connect: **Classify Message → Drop original message**

8) **Create the Data Table**
   - In n8n **Data Tables**, create a table named (example) **`mobile_payments`** with columns:
     - `reference` (string)
     - `payer` (string)
     - `phone` (string)
     - `amount` (number)
     - `date` (string)
     - `time` (string)
     - **Important:** also add `groupId` (string) if you plan to filter by it (the workflow uses it extensively even though it’s not listed in the Save payment schema block).
   - Note the table ID/name for nodes.

9) **Add Code node: “Mpesa”**
   - Parse receipt with the provided regex and output:
     - `provider`, `reference`, `amount`, `payer`, `phone`, `groupId: $json.from`, `date`, `time`
   - Connect: **Route (mpesa) → Mpesa**

10) **Add Code node: “Airtel Money”**
   - Parse receipt and output same normalized fields (with `provider: 'airtel'`).
   - Connect: **Route (airtel) → Airtel Money**

11) **Add Data Table node: “Save payment”**
   - Operation: **Insert row**
   - Table: `mobile_payments`
   - Mapping: auto-map input fields to columns.
   - Connect:
     - **Mpesa → Save payment**
     - **Airtel Money → Save payment**

12) **Add Data Table node: “Fetch payments”**
   - Operation: **Get**
   - Filter by `groupId`
   - Recommended fix (so summary works reliably): use **one** condition:
     - `groupId = {{$json.from}}` (for summary requests)
     - or ensure `groupId` exists in all branches before fetching.
   - Connect:
     - **Save payment → Fetch payments**
     - **Route (summary) → Fetch payments**

13) **Add Function node: “Create Summary”**
   - Compute total and format message output as `json.message`.
   - Connect: **Fetch payments → Create Summary**

14) **Add Data Table node: “Delete payments”**
   - Operation: **Delete rows**
   - Filter by `groupId` (again, prefer a single consistent filter).
   - Connect: **Route (clean) → Delete payments**

15) **Add Function node: “Create Message”**
   - Output fixed message: “Payment collection campaign has been closed. Details deleted”
   - Connect: **Delete payments → Create Message**

16) **Add Merge node: “Merge”**
   - Configure to merge two inputs (defaults can work, but ensure it waits for both):
     - Input 0: metadata
     - Input 1: generated message
   - Connect:
     - **Drop original message → Merge (Input 0)**
     - **Create Summary → Merge (Input 1)**
     - **Create Message → Merge (Input 1)**

17) **Add Code node: “Build one object”**
   - Merge all input items’ JSON into one object and output a single item.
   - Connect: **Merge → Build one object**

18) **Add IF node: “Valid message ?”**
   - Conditions:
     - `message` exists
     - `message` is not empty
   - Connect: **Build one object → Valid message ?**

19) **Add Twilio node: “Send a WhatsApp message”**
   - Credentials: configure **Twilio API** credentials (Account SID + Auth Token) in n8n.
   - Enable WhatsApp mode (in node: `toWhatsapp: true`)
   - Set:
     - To: `{{$json.from.replace('whatsapp:', '')}}`
     - From: `{{$json.to.replace('whatsapp:', '')}}`
     - Message: `{{$json.message}}`
   - Connect: **Valid message ? (true) → Send a WhatsApp message**

20) **(Strongly recommended) Fix webhook response**
   - Either:
     - Add a **Respond to Webhook** node to immediately return `200 OK`, or
     - Change Webhook response mode away from `responseNode`.
   - This avoids Twilio retries/timeouts.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “Need Help? Join the Discord or ask in the Forum!” | Discord: https://discord.com/invite/XPKeKXeB7d ; Forum: https://community.n8n.io/ |
| Production recommendation: use SQL instead of Data Table | Mentioned in sticky note: “SQL node is recommended for production use” |
| Operational guidance: forward SMS receipts into WhatsApp / add Twilio number to the group | Described in sticky note “How to use” |
| Supported providers: MPESA and AIRTEL MONEY (as-is) | Described in sticky note “Works with MPESA and AIRTEL MONEY, for now.” |
| Important limitation: no edit function; “clean” deletes details | Described in sticky note “Clearing may be necessary… since we provide no edit function.” |