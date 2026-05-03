Send post-purchase email sequences with Postgres, Gmail and OpenAI

https://n8nworkflows.xyz/workflows/send-post-purchase-email-sequences-with-postgres--gmail-and-openai-13274


# Send post-purchase email sequences with Postgres, Gmail and OpenAI

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:** Automate a post‑purchase email sequence driven by a Postgres `orders` table:  
1) detect new orders, 2) immediately send an order confirmation email, 3) wait until delivery is confirmed (polling), 4) send AI-generated usage tips, 5) wait two weeks, 6) send AI-generated upsell recommendations.

**Typical use cases**
- E-commerce post-purchase onboarding and retention
- Delivery-based conditional follow-ups
- AI-personalized product education and cross-sell campaigns

### Logical blocks
**1.1 Order Detection & Confirmation**  
Schedule trigger → query new orders (last 2 minutes) → send “order confirmed” email.

**1.2 Delivery Status Check (Polling Loop)**  
Wait 7 days → re-select order row → if not delivered, wait 1 day and re-check until delivered.

**1.3 Product Usage Tips (AI)**
When delivered → OpenAI generates bullet tips → code formats into HTML list → Gmail sends tips email.

**1.4 Upsell Recommendations (AI)**
Wait 14 days → OpenAI generates suggestions (same prompt structure) → code formats HTML list → Gmail sends upsell email.

---

## 2. Block-by-Block Analysis

### 2.1 Block — Order Detection & Confirmation
**Overview:** Runs every 2 minutes, fetches newly created orders, and sends an immediate order confirmation email to each order’s email address.

**Nodes involved:**
- Schedule Trigger
- Execute a SQL query
- Order Placed Ack.

#### Node: Schedule Trigger
- **Type / role:** `n8n-nodes-base.scheduleTrigger` — workflow entry point on a timer.
- **Config:** Every **2 minutes** (`minutesInterval: 2`).
- **Input/Output:** No input. Outputs into **Execute a SQL query**.
- **Failure/edge cases:**
  - If workflow execution time exceeds interval, executions can overlap (depends on n8n concurrency settings).
  - Clock drift/timezone doesn’t matter here because the query uses `NOW()` on Postgres side.

#### Node: Execute a SQL query
- **Type / role:** `n8n-nodes-base.postgres` — executes raw SQL to fetch new orders.
- **Operation:** Execute Query
- **Query logic:**  
  Selects all rows from `orders` created in the last 2 minutes:
  - `WHERE created_at >= NOW() - INTERVAL '2 minute'`
  - `ORDER BY created_at ASC`
- **Input/Output:** Receives trigger tick; outputs one item per returned row to **Order Placed Ack.**
- **Key assumptions:**
  - `orders` table exists and has at least `created_at`.
  - Returned rows include fields later used (e.g., `order_id`, `email`, `client_name`, `product_name`, `estimated_delivery_date`, `delivered`).
- **Failure/edge cases:**
  - DB credential/connection errors, timeouts.
  - If `created_at` is not indexed, this can become slow at scale.
  - If n8n execution is delayed >2 minutes, you may miss orders unless you widen the window or track last processed timestamp.

#### Node: Order Placed Ack.
- **Type / role:** `n8n-nodes-base.gmail` — sends the confirmation email.
- **Config choices:**
  - **To:** `{{$json.email}}`
  - **Subject:** `"Your Order Is Confirmed "`
  - **HTML body:** Uses template variables:
    - `{{$json.client_name}}`
    - `{{$json.product_name}}`
    - `{{$json.estimated_delivery_date.split('T')[0]}}` (assumes ISO-like string)
- **Connections:** Input from **Execute a SQL query**; output to **Wait until product get deliver**.
- **Failure/edge cases:**
  - Gmail auth issues / revoked OAuth.
  - Invalid/missing `email`.
  - If `estimated_delivery_date` is not a string containing `T`, the `.split('T')[0]` expression can fail.

---

### 2.2 Block — Delivery Status Check (Polling Loop)
**Overview:** Waits 7 days after order confirmation, then checks if the order is delivered. If not delivered, waits 1 more day and checks again, repeating until delivery is true.

**Nodes involved:**
- Wait until product get deliver
- Select rows from a table
- If2
- Wait for a day

#### Node: Wait until product get deliver
- **Type / role:** `n8n-nodes-base.wait` — delays execution.
- **Config:** Wait **7 days**.
- **Connections:** Input from **Order Placed Ack.** → output to **Select rows from a table**.
- **Failure/edge cases:**
  - Wait nodes rely on n8n persistence; if the instance is misconfigured for long waits (DB, queue mode), resumed executions can be impacted.

#### Node: Select rows from a table
- **Type / role:** `n8n-nodes-base.postgres` — selects the latest order row by `order_id`.
- **Operation:** Select
- **Config choices:**
  - **Schema:** `public`
  - **Table:** `orders`
  - **Where clause:** `order_id = {{ $('Execute a SQL query').item.json.order_id }}`
- **Connections:** Inputs from **Wait until product get deliver** and **Wait for a day**; output to **If2**.
- **Key expressions/variables:**
  - Uses `$('Execute a SQL query').item.json.order_id` referencing the earlier node’s item.
- **Failure/edge cases:**
  - If the workflow returns multiple orders per run, cross-item referencing can be error-prone if not truly paired per item. n8n generally keeps item lineage, but node referencing by name should be validated in multi-item runs.
  - If `order_id` is not unique or missing, selection may return multiple/no rows.

#### Node: If2
- **Type / role:** `n8n-nodes-base.if` — branching based on delivery status.
- **Condition:** `{{ $json.delivered }} == true` (boolean equals).
- **Connections:**
  - **True** → **Message a model** (usage tips)
  - **False** → **Wait for a day** (continue polling)
- **Failure/edge cases:**
  - If `delivered` is NULL/string (e.g., `"true"`), strict boolean comparison may fail and route to false.
  - If Select returns 0 rows, `$json.delivered` will be undefined.

#### Node: Wait for a day
- **Type / role:** `n8n-nodes-base.wait` — delay between delivery checks.
- **Config:** Wait **1 day**.
- **Connections:** Input from **If2 (false)** → output back to **Select rows from a table** (loop).
- **Failure/edge cases:** Same long-wait persistence considerations as other Wait nodes.

---

### 2.3 Block — Product Usage Tips (AI)
**Overview:** Once delivery is confirmed, generates up to 5 concise bullet tips using OpenAI, formats them into HTML, and emails them to the customer.

**Nodes involved:**
- Message a model
- Format AI response3
- Send Tips to User

#### Node: Message a model
- **Type / role:** `@n8n/n8n-nodes-langchain.openAi` — calls OpenAI chat model through n8n LangChain integration.
- **Config choices:**
  - **Model:** `modelId` is not set in the JSON (must be selected in n8n UI).
  - **Prompt content:** Instructs:
    - No intro/closing, max 5 bullet points
    - Use `*` bullet only
    - One short sentence per bullet
    - Friendly, simple tone, no emojis, no headings, no newline chars
    - Product injected: `{{ $('Execute a SQL query').item.json.product_name }}`
- **Credentials:** OpenAI API credential required.
- **Connections:** Input from **If2 (true)** → output to **Format AI response3**.
- **Failure/edge cases:**
  - Missing model selection will cause runtime error.
  - Model may ignore “no newline chars”; output parsing depends on `* ` delimiter.
  - Rate limits / API errors.

#### Node: Format AI response3
- **Type / role:** `n8n-nodes-base.code` — transforms AI output into safe HTML list.
- **Logic:**
  - Reads `item.json.output` (expects AI text in `output`)
  - Splits by `"* "` and trims
  - Wraps items in `<ul><li>…</li></ul>`
  - **Preserves original fields** via spread: `...original`
  - Adds `formattedOutput`
- **Connections:** Input from **Message a model** → output to **Send Tips to User**.
- **Failure/edge cases:**
  - If AI output does not contain `* ` bullets, list may become empty or include the whole text as one item.
  - If the OpenAI node returns a different property name than `output`, formatting will fail (depends on node version/response settings).

#### Node: Send Tips to User
- **Type / role:** `n8n-nodes-base.gmail` — sends “getting started” email with formatted tips.
- **Config choices:**
  - **To:** `{{ $('Execute a SQL query').item.json.email }}`
  - **Subject:** `Getting Started with Your {{ $('Execute a SQL query').item.json.product_name }}`
  - **HTML body:** Inserts `{{$json.formattedOutput}}` (from Code node).
  - Greeting line is hardcoded: `Hi <strong>Devansh</strong>,` (not dynamic).
- **Connections:** Input from **Format AI response3** → output to **Wait for 2 weeks**.
- **Failure/edge cases:**
  - Hardcoded name can be incorrect (should likely use `client_name`).
  - Gmail auth / invalid email address.
  - If `formattedOutput` missing, email will show empty area.

---

### 2.4 Block — Upsell Recommendations (AI)
**Overview:** After two weeks, generates complementary product suggestions via OpenAI, formats them, and sends an upsell email.

**Nodes involved:**
- Wait for 2 weeks
- Message a model1
- Code in JavaScript1
- Send Tips to User1

#### Node: Wait for 2 weeks
- **Type / role:** `n8n-nodes-base.wait` — delays for follow-up.
- **Config:** Wait **14 days**.
- **Connections:** Input from **Send Tips to User** → output to **Message a model1**.
- **Failure/edge cases:** Long-wait persistence considerations.

#### Node: Message a model1
- **Type / role:** `@n8n/n8n-nodes-langchain.openAi` — OpenAI generation for follow-up content.
- **Config:** Prompt text is effectively identical to the tips prompt (still “usage tips”), but used as “upsell recommendations” in the workflow notes. Product injected from the original order: `{{ $('Execute a SQL query').item.json.product_name }}`
- **Model:** `modelId` not set (must select).
- **Connections:** Input from **Wait for 2 weeks** → output to **Code in JavaScript1**.
- **Failure/edge cases:** Same as first OpenAI node; also prompt mismatch (it asks for “usage tips” but the email is “recommendations”).

#### Node: Code in JavaScript1
- **Type / role:** `n8n-nodes-base.code` — formats AI output into HTML list (same pattern).
- **Logic:** Splits on `'* '`, builds `<ul>`.
- **Connections:** Input from **Message a model1** → output to **Send Tips to User1**.
- **Failure/edge cases:** Same parsing sensitivity to bullet format and output property name.

#### Node: Send Tips to User1
- **Type / role:** `n8n-nodes-base.gmail` — sends upsell/recommendation email.
- **Config choices:**
  - **To:** `{{ $('Execute a SQL query').item.json.email }}`
  - **Subject:** `You Might Love This with Your {{ $('Execute a SQL query').item.json.product_name }}`
  - **Greeting:** dynamic `client_name`
  - **Body:** inserts `{{$json.formattedOutput}}`
- **Connections:** Final node (no outputs).
- **Failure/edge cases:** Gmail auth, invalid email, missing `client_name`/`formattedOutput`.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Schedule Trigger | Schedule Trigger | Polling entry point (every 2 minutes) | — | Execute a SQL query | ## Step 1 — Order Detection & Confirmation / Fetches new orders and sends an order confirmation email to the customer. |
| Execute a SQL query | Postgres | Fetch new orders created within the last 2 minutes | Schedule Trigger | Order Placed Ack. | ## Step 1 — Order Detection & Confirmation / Fetches new orders and sends an order confirmation email to the customer. |
| Order Placed Ack. | Gmail | Send immediate order confirmation | Execute a SQL query | Wait until product get deliver | ## Step 1 — Order Detection & Confirmation / Fetches new orders and sends an order confirmation email to the customer. |
| Wait until product get deliver | Wait | Initial delay before checking delivery | Order Placed Ack. | Select rows from a table | ## Step 2 — Delivery Status Check / Waits for delivery and rechecks order status until delivered. |
| Select rows from a table | Postgres | Re-fetch order row by order_id for current status | Wait until product get deliver; Wait for a day | If2 | ## Step 2 — Delivery Status Check / Waits for delivery and rechecks order status until delivered. |
| If2 | IF | Branch based on `delivered == true` | Select rows from a table | Message a model (true); Wait for a day (false) | ## Step 2 — Delivery Status Check / Waits for delivery and rechecks order status until delivered. |
| Wait for a day | Wait | Delay between delivery polling checks | If2 (false) | Select rows from a table | ## Step 2 — Delivery Status Check / Waits for delivery and rechecks order status until delivered. |
| Message a model | OpenAI (LangChain) | Generate post-delivery usage tips | If2 (true) | Format AI response3 | ## Step 3 — Product Usage Tips (AI) / Generates and emails AI-based usage tips after delivery. |
| Format AI response3 | Code | Convert `*` bullets into HTML `<ul>` | Message a model | Send Tips to User | ## Step 3 — Product Usage Tips (AI) / Generates and emails AI-based usage tips after delivery. |
| Send Tips to User | Gmail | Email the formatted usage tips | Format AI response3 | Wait for 2 weeks | ## Step 3 — Product Usage Tips (AI) / Generates and emails AI-based usage tips after delivery. |
| Wait for 2 weeks | Wait | Delay before upsell follow-up | Send Tips to User | Message a model1 | ## Step 4 — Upsell Recommendations (AI) / Waits two weeks and sends AI-generated complementary product suggestions. |
| Message a model1 | OpenAI (LangChain) | Generate follow-up bullets (used as upsell content) | Wait for 2 weeks | Code in JavaScript1 | ## Step 4 — Upsell Recommendations (AI) / Waits two weeks and sends AI-generated complementary product suggestions. |
| Code in JavaScript1 | Code | Convert `*` bullets into HTML `<ul>` | Message a model1 | Send Tips to User1 | ## Step 4 — Upsell Recommendations (AI) / Waits two weeks and sends AI-generated complementary product suggestions. |
| Send Tips to User1 | Gmail | Email the formatted upsell recommendations | Code in JavaScript1 | — | ## Step 4 — Upsell Recommendations (AI) / Waits two weeks and sends AI-generated complementary product suggestions. |
| Sticky Note | Sticky Note | Documentation block | — | — | ## Order Placed → Delivery-Based Upsell Automation (full description + setup steps in canvas) |
| Step -1 Trigger & Order Detection | Sticky Note | Documentation for Step 1 | — | — | ## Step 1 — Order Detection & Confirmation (canvas note) |
| Sticky Note1 | Sticky Note | Documentation for Step 2 | — | — | ## Step 2 — Delivery Status Check (canvas note) |
| Sticky Note2 | Sticky Note | Documentation for Step 3 | — | — | ## Step 3 — Product Usage Tips (AI) (canvas note) |
| Sticky Note3 | Sticky Note | Documentation for Step 4 | — | — | ## Step 4 — Upsell Recommendations (AI) (canvas note) |

---

## 4. Reproducing the Workflow from Scratch

1) **Create Schedule Trigger**
   - Add node: **Schedule Trigger**
   - Set rule: **Every 2 minutes**

2) **Add Postgres node to fetch new orders**
   - Add node: **Postgres** → Operation: **Execute Query**
   - Configure Postgres credentials to your database
   - SQL:
     - `SELECT * FROM orders WHERE created_at >= NOW() - INTERVAL '2 minute' ORDER BY created_at ASC;`
   - Connect: **Schedule Trigger → Execute a SQL query**

3) **Add Gmail node: order confirmation**
   - Add node: **Gmail** → Operation: **Send**
   - Configure Gmail OAuth2 credentials (Google project or n8n Gmail OAuth)
   - To: `{{$json.email}}`
   - Subject: `Your Order Is Confirmed`
   - Body: HTML using expressions for `client_name`, `product_name`, and `estimated_delivery_date`
   - Connect: **Execute a SQL query → Order Placed Ack.**

4) **Add Wait (7 days)**
   - Add node: **Wait**
   - Unit: **Days**, Amount: **7**
   - Connect: **Order Placed Ack. → Wait until product get deliver**

5) **Add Postgres select-by-order_id**
   - Add node: **Postgres** → Operation: **Select**
   - Schema: `public`, Table: `orders`
   - Where: `order_id` equals `{{ $('Execute a SQL query').item.json.order_id }}`
   - Connect: **Wait until product get deliver → Select rows from a table**

6) **Add IF delivered?**
   - Add node: **IF**
   - Condition: Boolean equals
     - Left: `{{$json.delivered}}`
     - Right: `true`
   - Connect: **Select rows from a table → If2**

7) **Add Wait (1 day) for re-check loop**
   - Add node: **Wait**
   - Unit: **Days**, Amount: **1**
   - Connect: **If2 (false) → Wait for a day → Select rows from a table** (creates the polling loop)

8) **Add OpenAI node for usage tips**
   - Add node: **OpenAI (LangChain) – Message a model**
   - Select a **modelId** (e.g., GPT-4.x / GPT-4o-mini, depending on availability)
   - Add OpenAI API credentials
   - Prompt content: enforce `*` bullets, max 5, no intro/closing, and inject product name:
     - `And Product is : {{ $('Execute a SQL query').item.json.product_name }}`
   - Connect: **If2 (true) → Message a model**

9) **Add Code node to format bullets into HTML**
   - Add node: **Code** (JavaScript)
   - Implement:
     - Read `item.json.output`
     - Split on `"* "`
     - Build `<ul><li>…</li></ul>`
     - Preserve original JSON and add `formattedOutput`
   - Connect: **Message a model → Format AI response3**

10) **Add Gmail node to send tips**
   - Add node: **Gmail** → Send
   - To: `{{ $('Execute a SQL query').item.json.email }}`
   - Subject: `Getting Started with Your {{ $('Execute a SQL query').item.json.product_name }}`
   - Body: insert `{{$json.formattedOutput}}`
   - Connect: **Format AI response3 → Send Tips to User**

11) **Add Wait (14 days)**
   - Add node: **Wait**
   - Unit: **Days**, Amount: **14**
   - Connect: **Send Tips to User → Wait for 2 weeks**

12) **Add OpenAI node for upsell content**
   - Add node: **OpenAI (LangChain) – Message a model**
   - Select **modelId**
   - Use a prompt (ideally adapted for “complementary product suggestions”; in the provided workflow it mirrors the tips prompt)
   - Connect: **Wait for 2 weeks → Message a model1**

13) **Add Code node to format upsell bullets**
   - Add node: **Code** (JavaScript)
   - Same HTML list formatting, preserving original data, output to `formattedOutput`
   - Connect: **Message a model1 → Code in JavaScript1**

14) **Add Gmail node to send upsell email**
   - Add node: **Gmail** → Send
   - To: `{{ $('Execute a SQL query').item.json.email }}`
   - Subject: `You Might Love This with Your {{ $('Execute a SQL query').item.json.product_name }}`
   - Body: include `{{$json.formattedOutput}}` and `client_name`
   - Connect: **Code in JavaScript1 → Send Tips to User1**

15) **(Optional but recommended) Add/adjust data requirements**
   - Ensure `orders` includes at least:
     - `order_id` (unique), `email`, `client_name`, `product_name`, `created_at`, `estimated_delivery_date`, `delivered` (boolean)

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Order Placed → Delivery-Based Upsell Automation: confirms new orders, waits for delivery with daily polling until delivered, sends AI tips, waits two weeks, then sends AI upsell recommendations. Setup includes Postgres + Gmail + AI credentials and adjusting wait durations. | From the large canvas sticky note (workflow description and setup steps). |
| Step 1 — Order Detection & Confirmation: Fetches new orders and sends an order confirmation email to the customer. | Canvas note for Step 1. |
| Step 2 — Delivery Status Check: Waits for delivery and rechecks order status until delivered. | Canvas note for Step 2. |
| Step 3 — Product Usage Tips (AI): Generates and emails AI-based usage tips after delivery. | Canvas note for Step 3. |
| Step 4 — Upsell Recommendations (AI): Waits two weeks and sends AI-generated complementary product suggestions. | Canvas note for Step 4. |