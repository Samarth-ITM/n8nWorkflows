Send AI-powered post-purchase emails and WhatsApp messages from PostgreSQL to Google Sheets with Groq

https://n8nworkflows.xyz/workflows/send-ai-powered-post-purchase-emails-and-whatsapp-messages-from-postgresql-to-google-sheets-with-groq-12673


# Send AI-powered post-purchase emails and WhatsApp messages from PostgreSQL to Google Sheets with Groq

## 1. Workflow Overview

**Purpose:** Automatically detect **completed orders** in PostgreSQL, fetch associated customer/product/payment details, generate a **personalized post‑purchase message using Groq (LLM)**, send it via **Gmail** and **WhatsApp**, then **log the communication** to **Google Sheets**.

**Primary use cases:**
- Post-purchase customer engagement for SaaS/e-commerce flows
- Payment-aware messaging (thank-you vs. polite reminder)
- Centralized logging of outbound communications

### 1.1 Trigger & Order Intake (PostgreSQL)
Listens for order activity, queries completed orders, and batches through results.

### 1.2 Enrichment (Join order + user + product + payment)
Loads full context needed to generate high-quality personalized messaging.

### 1.3 AI Message Generation (Groq via LangChain Agent)
Builds a structured prompt with customer/order details and generates message text with strict rules.

### 1.4 Formatting, Sending (Gmail + WhatsApp) & Logging (Google Sheets)
Formats AI output into channel-specific templates, sends, flags status, appends a row in Sheets, and waits before continuing the loop.

---

## 2. Block-by-Block Analysis

### Block 1 — Trigger & Prepare Order List
**Overview:** Starts on PostgreSQL table events, retrieves all completed orders, then iterates safely through them using batching.

**Nodes involved:**
- Postgres Trigger1
- Execute a SQL query2
- Loop Over Items1

#### Node: Postgres Trigger1
- **Type / role:** `Postgres Trigger` — event-based entry point from PostgreSQL.
- **Configuration (interpreted):**
  - Schema: `public`
  - Table: `orders`
  - Triggers whenever the node’s configured DB trigger fires (exact operation type not specified in JSON beyond table selection).
- **Inputs/Outputs:**
  - **Output →** Execute a SQL query2
- **Version notes:** TypeVersion `1` (older trigger node behavior; ensure DB permissions to create/listen to triggers).
- **Failure/edge cases:**
  - Missing/invalid Postgres credentials
  - DB user lacks permission to create/subscribe to triggers
  - Trigger fires on changes unrelated to “completed” status (the workflow filters later)

#### Node: Execute a SQL query2
- **Type / role:** `Postgres` (Execute Query) — fetch IDs of completed orders.
- **Key config:**
  - Query:
    ```sql
    SELECT id
    FROM orders
    WHERE order_status = 'completed'
    ```
- **Inputs/Outputs:**
  - **Input ←** Postgres Trigger1
  - **Output →** Loop Over Items1
- **Version notes:** TypeVersion `2.6`
- **Failure/edge cases:**
  - Large result set could increase runtime; batching helps but initial query still returns all completed orders
  - `order_status` values must match exactly (`completed`)

#### Node: Loop Over Items1
- **Type / role:** `Split in Batches` — iterates through order IDs.
- **Key config:** default batching options (batch size not explicitly set in JSON).
- **Connections:**
  - **Input ←** Execute a SQL query2
  - **Main output (index 1) →** Execute a SQL query3 (this workflow uses the *second* output for processing)
  - **Main output (index 0)** is unused here
- **Version notes:** TypeVersion `3`
- **Failure/edge cases:**
  - Miswiring risk: processing happens on output **1**; if you connect output 0 by mistake, nothing proceeds
  - If batch size defaults are unsuitable, may be slow or overload downstream APIs

---

### Block 2 — Enrich Order Data (Order + Product + User + Payment)
**Overview:** For each order ID, loads all the details required to personalize the message and decide tone based on payment status.

**Nodes involved:**
- Execute a SQL query3

#### Node: Execute a SQL query3
- **Type / role:** `Postgres` (Execute Query) — fetches detailed row for the current order.
- **Key config:**
  - Query joins `orders`, `products`, `users`, and optional `payment_history`:
    - Uses the current item’s `id`:
      - `WHERE o.id = {{$json["id"]}};`
  - Selected fields include:
    - order: `order_id`, `ordered_at`, `order_status`
    - user: `user_id`, `name`, `email`, `phone`
    - product: `product_id`, `product_name`
    - payment: `payment_status` (from `payment_history`, left join)
- **Connections:**
  - **Input ←** Loop Over Items1 (output index 1)
  - **Output →** AI Agent1 (main)
  - **Output →** Merge1 (main, index 1)
- **Version notes:** TypeVersion `2.6`
- **Failure/edge cases:**
  - If `payment_history` has multiple rows per order, this join may return multiple items per order (duplicate messages/logs). Consider aggregating or limiting (e.g., latest payment record).
  - If no payment record exists, `payment_status` may be `null` → downstream logic should handle (agent rules mention “unpaid/pending/paid” only; null can produce ambiguous tone).
  - If phone/email is missing, send nodes may fail.

---

### Block 3 — AI Message Generation (Groq + Agent)
**Overview:** Creates a structured prompt from the enriched DB row and generates a post-purchase message, with strict rules for personalization and payment-aware tone.

**Nodes involved:**
- Groq Chat Model1
- AI Agent1
- Merge1

#### Node: Groq Chat Model1
- **Type / role:** LangChain Groq Chat Model — provides the LLM for the agent.
- **Key config:**
  - Model: `llama-3.3-70b-versatile`
- **Connections:**
  - **AI languageModel output →** AI Agent1 (special `ai_languageModel` connection)
- **Version notes:** TypeVersion `1`
- **Failure/edge cases:**
  - Missing/invalid Groq API credentials
  - Rate limits / timeouts
  - Model availability changes

#### Node: AI Agent1
- **Type / role:** LangChain Agent — generates final message text.
- **Key config:**
  - **User message template** includes:
    - Customer name/email/phone
    - Product name
    - Order status
    - Payment status
  - **System message rules (important):**
    - Must personalize using name
    - Tone depends on payment status
    - Paid: thanks + next steps + soft upsell
    - Unpaid/pending: polite reminder, no upsell
    - No invented data
    - Output only final message text
- **Connections:**
  - **Main input ←** Execute a SQL query3
  - **Main output →** Merge1 (index 0)
  - **LLM input ←** Groq Chat Model1 (ai_languageModel)
- **Version notes:** TypeVersion `3`
- **Failure/edge cases:**
  - If `payment_status` is `null` or unexpected value, agent may not follow desired branch cleanly
  - If upstream returns multiple rows, agent runs multiple times
  - Agent output field used later is `json.output` (if node outputs a different field in some versions/configs, formatting code breaks)

#### Node: Merge1
- **Type / role:** `Merge` — combines AI output and DB data so formatting can access both.
- **Key behavior here:** The workflow sends:
  - AI Agent output into **Merge input 0**
  - Postgres detailed row into **Merge input 1**
- **Connections:**
  - **Input 0 ←** AI Agent1
  - **Input 1 ←** Execute a SQL query3
  - **Output →** Format AI response1
- **Version notes:** TypeVersion `3.2`
- **Failure/edge cases:**
  - Merge mode isn’t explicitly shown; default behavior can vary by node/version. The downstream code expects **both items to be present** in `$input.all()`.
  - If either branch fails, formatter may output empty messages.

---

### Block 4 — Format, Send, Flag, Log, Wait/Loop
**Overview:** Builds channel-ready email and WhatsApp bodies, sends both messages, marks status flags, logs to Google Sheets, then waits and loops back to continue batch processing.

**Nodes involved:**
- Format AI response1
- Send a message1 (Gmail)
- Send message1 (WhatsApp)
- Edit Fields1
- Append row in sheet1
- Wait1

#### Node: Format AI response1
- **Type / role:** `Code` — transforms merged inputs into clean message payloads.
- **Key logic/expressions:**
  - Reads all incoming items: `const items = $input.all();`
  - Detects:
    - AI item: `items.find(i => i.json.output)`
    - Data item: `items.find(i => i.json.name)`
  - Normalizes AI text:
    - trims + collapses excessive newlines
  - Produces:
    - `email_message` with greeting + support signature
    - `whatsapp_message` with short template (includes emoji in template)
  - Returns **one item** containing customer/product/contact/status fields + both messages.
- **Connections:**
  - **Input ←** Merge1
  - **Output →** Send a message1 (Gmail)
  - **Output →** Send message1 (WhatsApp)
- **Version notes:** TypeVersion `2`
- **Failure/edge cases:**
  - If AI Agent output key is not `output`, `aiItem` becomes undefined → empty message
  - If DB row lacks `name`, `email`, or `phone`, downstream sending can fail
  - The WhatsApp template includes emoji; remove if your channel/provider rejects it

#### Node: Send a message1 (Gmail)
- **Type / role:** `Gmail` — sends email to customer.
- **Key config:**
  - To: `{{$json.email}}`
  - Subject: `Product Purchase Completion`
  - Body: `{{$json.email_message}}`
- **Connections:**
  - **Input ←** Format AI response1
  - **Output →** Edit Fields1
- **Version notes:** TypeVersion `2.2`
- **Failure/edge cases:**
  - Gmail OAuth not configured / expired token
  - Invalid recipient email
  - Provider rate limits
  - Note: a `webhookId` is present in JSON but Gmail node typically relies on OAuth credentials; ensure correct Gmail credential setup in n8n.

#### Node: Send message1 (WhatsApp)
- **Type / role:** `WhatsApp` — sends WhatsApp message to customer phone.
- **Key config:**
  - Operation: `send`
  - Recipient phone: `{{$json.phone}}`
  - Text body: `{{$json.whatsapp_message}}`
- **Connections:**
  - **Input ←** Format AI response1
  - **Output →** Edit Fields1
- **Version notes:** TypeVersion `1.1`
- **Failure/edge cases:**
  - WhatsApp provider credentials not configured (depends on n8n WhatsApp node integration)
  - Phone format requirements (E.164 usually, e.g., `+15551234567`)
  - Template/message restrictions depending on provider (session window, approved templates)

#### Node: Edit Fields1
- **Type / role:** `Set` — prepares a clean logging payload and sets send status flags.
- **Key config:**
  - Copies fields from `Format AI response1` using expressions like:
    - `={{ $('Format AI response1').item.json.customer_name }}`
  - Sets:
    - `Email_Send = "True"`
    - `Whatsapp_Message_Send = "True"`
- **Connections:**
  - **Inputs ←** Send a message1 AND Send message1 (both feed into it)
  - **Output →** Append row in sheet1
- **Version notes:** TypeVersion `3.4`
- **Failure/edge cases:**
  - If either send node errors, this node may never run (unless you enable “Continue On Fail” on send nodes)
  - Flags are always set to `"True"` regardless of actual delivery result; consider reading send node response or using error branches

#### Node: Append row in sheet1
- **Type / role:** `Google Sheets` — appends a row for tracking.
- **Key config:**
  - Operation: Append
  - Document ID: `YOUR_GOOGLE_SHEETS_DOCUMENT_ID`
  - Sheet tab: selected by internal sheet ID `2092498842` (with cached name/url)
  - Columns schema shown includes many fields marked “removed”; mapping is set to “define below” but the actual mapped fields are not explicitly listed in JSON values (likely configured in UI).
- **Connections:**
  - **Input ←** Edit Fields1
  - **Output →** Wait1
- **Version notes:** TypeVersion `4.7`
- **Failure/edge cases:**
  - Google OAuth credential missing/expired
  - Sheet permissions (must have edit access)
  - Column mapping mismatch: if the sheet columns don’t match what the node expects, append may fail or write blanks

#### Node: Wait1
- **Type / role:** `Wait` — pauses the workflow then continues.
- **Key config:** no explicit wait duration shown (default wait behavior depends on how node is configured in UI; JSON has empty parameters).
- **Connections:**
  - **Input ←** Append row in sheet1
  - **Output →** Loop Over Items1 (to continue processing next batch)
- **Version notes:** TypeVersion `1.1`
- **Failure/edge cases:**
  - If configured as “Wait for webhook” in UI, it will halt until externally resumed (risky for automation)
  - If configured as “Wait some time”, ensure it’s appropriate to avoid excessive delays or API bursts

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Postgres Trigger1 | postgresTrigger | Entry trigger on `public.orders` | — | Execute a SQL query2 | ## AI-Powered Post-Purchase Journey (Email + WhatsApp Automation)<br>Automatically sends AI-generated post-purchase messages when an order is marked **completed**. The workflow personalizes communication, delivers it via Email and WhatsApp, and logs results for tracking.<br><br>### How it works<br>- Triggers on completed orders in the `orders` table<br>- Fetches customer, product, and payment data<br>- Generates a personalized AI message<br>- Sends Email and WhatsApp notifications<br>- Logs communication in Google Sheets<br><br>### Setup steps<br>1. Connect PostgreSQL<br>2. Configure the Postgres Trigger<br>3. Connect AI model (Groq / LLM)<br>4. Connect Gmail and WhatsApp<br>5. Configure Google Sheets logging<br>6. Activate and test the workflow<br><br>## Step 1: Fetch & Prepare Orders<br>Detect completed orders, loop safely through them, and retrieve full customer, product, and payment details. Send structured data to the AI Agent. |
| Execute a SQL query2 | postgres | Query completed order IDs | Postgres Trigger1 | Loop Over Items1 | (same as above) |
| Loop Over Items1 | splitInBatches | Batch iteration over order IDs | Execute a SQL query2, Wait1 | Execute a SQL query3 | (same as above) |
| Execute a SQL query3 | postgres | Enrich each order with user/product/payment details | Loop Over Items1 | AI Agent1, Merge1 | (same as above) |
| Groq Chat Model1 | lmChatGroq | LLM backend for agent (Groq) | — | AI Agent1 (ai_languageModel) | (same as above) |
| AI Agent1 | langchain.agent | Generate payment-aware post-purchase message | Execute a SQL query3, Groq Chat Model1 | Merge1 | (same as above) |
| Merge1 | merge | Combine DB context + AI output for formatting | AI Agent1, Execute a SQL query3 | Format AI response1 | ## Step 2: Send Messages & Log Data<br>Format AI output for Email and WhatsApp, send messages via connected channels, flag delivery status, and log post-purchase communication into Google Sheets. |
| Format AI response1 | code | Build email/WhatsApp message bodies + normalized payload | Merge1 | Send a message1, Send message1 | (same as above) |
| Send a message1 | gmail | Send email via Gmail | Format AI response1 | Edit Fields1 | (same as above) |
| Send message1 | whatsApp | Send WhatsApp message | Format AI response1 | Edit Fields1 | (same as above) |
| Edit Fields1 | set | Prepare logging payload + set “sent” flags | Send a message1, Send message1 | Append row in sheet1 | (same as above) |
| Append row in sheet1 | googleSheets | Append tracking row to Google Sheets | Edit Fields1 | Wait1 | (same as above) |
| Wait1 | wait | Pause/throttle, then continue loop | Append row in sheet1 | Loop Over Items1 | (same as above) |
| Sticky Note3 | stickyNote | Documentation / context | — | — | ## AI-Powered Post-Purchase Journey (Email + WhatsApp Automation)… |
| Sticky Note4 | stickyNote | Documentation / block label | — | — | ## Step 1: Fetch & Prepare Orders… |
| Sticky Note5 | stickyNote | Documentation / block label | — | — | ## Step 2: Send Messages & Log Data… |

---

## 4. Reproducing the Workflow from Scratch

1. **Create credentials**
   1) **PostgreSQL credential** in n8n (host, db, user, password, SSL as needed).  
   2) **Groq credential** for the Groq LangChain node (API key).  
   3) **Gmail OAuth2 credential** (connect the sending mailbox).  
   4) **WhatsApp credential** required by your n8n WhatsApp node/provider (varies by integration).  
   5) **Google Sheets OAuth2 credential** with edit access to the target spreadsheet.

2. **Add “Postgres Trigger” node**
   - Schema: `public`
   - Table: `orders`
   - Connect it to the next node.

3. **Add “Postgres” node (Execute Query) — “Execute a SQL query2”**
   - Operation: *Execute Query*
   - SQL:
     - `SELECT id FROM orders WHERE order_status = 'completed'`
   - Connect from **Postgres Trigger → Execute a SQL query2**.

4. **Add “Split in Batches” node — “Loop Over Items1”**
   - Keep defaults or set a batch size (recommended if you expect many orders).
   - Connect **Execute a SQL query2 → Loop Over Items1**.
   - Important: connect the workflow processing chain from **output 1** (second output) of Split in Batches, matching the provided workflow wiring.

5. **Add “Postgres” node (Execute Query) — “Execute a SQL query3”**
   - Operation: *Execute Query*
   - SQL (use expression for current item ID):
     - `WHERE o.id = {{$json["id"]}};`
   - Connect **Loop Over Items1 (output 1) → Execute a SQL query3**.

6. **Add Groq LLM node — “Groq Chat Model1”**
   - Node: *Groq Chat Model*
   - Model: `llama-3.3-70b-versatile`
   - Select Groq credentials.

7. **Add LangChain “AI Agent” node — “AI Agent1”**
   - Prompt type: “define”
   - System message: paste the rules from the workflow (personalization, paid/unpaid behavior, no invented data, output only message).
   - User text: template including `{{$json["name"]}}`, `{{$json["email"]}}`, `{{$json["phone"]}}`, `{{$json["product_name"]}}`, `{{$json["order_status"]}}`, `{{$json["payment_status"]}}`.
   - Connect:
     - **Execute a SQL query3 → AI Agent1** (main)
     - **Groq Chat Model1 → AI Agent1** via the **ai_languageModel** connection type.

8. **Add “Merge” node — “Merge1”**
   - Connect:
     - **AI Agent1 → Merge1** (input 0)
     - **Execute a SQL query3 → Merge1** (input 1)
   - Leave merge settings at default only if it yields two items to the next step (the formatter expects both).

9. **Add “Code” node — “Format AI response1”**
   - Paste the provided JS logic that:
     - finds `json.output` (AI) and `json.name` (DB)
     - builds `email_message` and `whatsapp_message`
     - outputs a single consolidated item.
   - Connect **Merge1 → Format AI response1**.

10. **Add Gmail node — “Send a message1”**
   - Resource/Operation: send email
   - To: `={{ $json.email }}`
   - Subject: `Product Purchase Completion`
   - Message: `={{ $json.email_message }}`
   - Connect **Format AI response1 → Send a message1**.

11. **Add WhatsApp node — “Send message1”**
   - Operation: `send`
   - Recipient phone: `={{ $json.phone }}`
   - Text body: `={{ $json.whatsapp_message }}`
   - Connect **Format AI response1 → Send message1**.

12. **Add “Set” node — “Edit Fields1”**
   - Add fields (string) using expressions referencing the formatter node:
     - `customer_name`, `product_name`, `email`, `phone`, `payment_status`, `order_status`
   - Add flags:
     - `Email_Send = "True"`
     - `Whatsapp_Message_Send = "True"`
   - Connect **Send a message1 → Edit Fields1** and **Send message1 → Edit Fields1**.

13. **Add Google Sheets node — “Append row in sheet1”**
   - Operation: *Append*
   - Select the Spreadsheet (Document ID) and Sheet tab.
   - Map columns to fields from **Edit Fields1** (at minimum: customer_name, email, product_name, payment_status, plus flags if you add columns for them).
   - Connect **Edit Fields1 → Append row in sheet1**.

14. **Add “Wait” node — “Wait1”**
   - Configure as either:
     - “Wait for X time” (recommended for throttling), or
     - “Wait for webhook” (only if you explicitly want manual/external resume)
   - Connect **Append row in sheet1 → Wait1**.

15. **Close the loop**
   - Connect **Wait1 → Loop Over Items1** to continue processing the next batch.

16. **Activate and test**
   - Ensure an order transitions to `order_status='completed'`.
   - Confirm:
     - Postgres trigger fires
     - AI produces output in `output`
     - Gmail and WhatsApp send successfully
     - Sheet row is appended per processed order

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “AI-Powered Post-Purchase Journey (Email + WhatsApp Automation)… Setup steps… Activate and test the workflow” | Sticky Note content embedded in workflow canvas |
| “Step 1: Fetch & Prepare Orders…” | Sticky Note block label for PostgreSQL intake/enrichment |
| “Step 2: Send Messages & Log Data…” | Sticky Note block label for formatting/sending/logging |

