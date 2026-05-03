Send post-purchase emails from Postgres with Gmail and Groq AI

https://n8nworkflows.xyz/workflows/send-post-purchase-emails-from-postgres-with-gmail-and-groq-ai-14144


# Send post-purchase emails from Postgres with Gmail and Groq AI

# 1. Workflow Overview

This workflow automates post-purchase customer communication for orders stored in PostgreSQL. It detects new orders on a schedule, sends an order confirmation email through Gmail, waits for the estimated delivery period, repeatedly checks whether the order has actually been delivered, then uses Groq-powered AI agents to generate two follow-up emails: product usage tips shortly after delivery and complementary product recommendations two weeks later.

The overall purpose is to support customer onboarding and upsell after purchase with minimal manual effort.

## 1.1 Order Detection & Confirmation

A scheduled trigger runs every 2 minutes, queries the `orders` table for recently created orders, and sends an immediate confirmation email to each customer.

## 1.2 Delivery Wait & Delivery Status Recheck Loop

After the confirmation email, the workflow waits 7 days, checks the order’s current `delivered` flag in PostgreSQL, and if it is not yet delivered, waits 1 day and checks again. This loop continues until delivery is confirmed.

## 1.3 AI-Generated Product Usage Tips

Once delivery is confirmed, an AI agent is intended to generate short product usage tips based on the purchased product. A code node converts the AI text into HTML list markup, and a Gmail node sends the formatted tips to the customer.

## 1.4 AI-Generated Complementary Product Recommendations

Two weeks after the tips email, another AI agent is intended to generate complementary product suggestions. A second code node formats the AI response as HTML, and another Gmail node sends the recommendations email.

## 1.5 Documentation / In-Canvas Notes

Several sticky notes describe the workflow’s intent, logical steps, and setup guidance directly inside the n8n canvas.

---

# 2. Block-by-Block Analysis

## 2.1 Order Detection & Confirmation

### Overview
This block identifies newly created orders and sends a confirmation email immediately after detection. It is the workflow’s entry point and establishes the customer/order context used by downstream steps.

### Nodes Involved
- Schedule Trigger
- Execute a SQL query
- Order Placed Ack.

### Node Details

#### Schedule Trigger
- **Type and technical role:** `n8n-nodes-base.scheduleTrigger`; starts the workflow automatically on a recurring schedule.
- **Configuration choices:** Configured to run every 2 minutes.
- **Key expressions or variables used:** None.
- **Input and output connections:** No incoming connection; outputs to **Execute a SQL query**.
- **Version-specific requirements:** `typeVersion: 1.3`.
- **Edge cases or potential failure types:**
  - Workflow must be active for the trigger to run.
  - Very frequent polling may cause duplicate processing if the SQL window overlaps and processed orders are not marked elsewhere.
  - Timezone settings in n8n may affect operational expectations, though the SQL query itself uses database server time.
- **Sub-workflow reference:** None.

#### Execute a SQL query
- **Type and technical role:** `n8n-nodes-base.postgres`; executes a custom SQL query against PostgreSQL.
- **Configuration choices:** Uses `executeQuery` with:
  ```sql
  SELECT *
  FROM orders
  WHERE created_at >= NOW() - INTERVAL '2 minute'
  ORDER BY created_at ASC;
  ```
  This retrieves orders created in the last 2 minutes.
- **Key expressions or variables used:** None inside the SQL itself beyond PostgreSQL `NOW()`.
- **Input and output connections:** Input from **Schedule Trigger**; output to **Order Placed Ack.**
- **Version-specific requirements:** `typeVersion: 2.6`.
- **Edge cases or potential failure types:**
  - Requires valid PostgreSQL credentials.
  - If polling every 2 minutes with `>= NOW() - INTERVAL '2 minute'`, the same order can be picked up more than once across close executions.
  - If `created_at` precision or time drift exists, records may be missed or duplicated.
  - Query assumes an `orders` table and `created_at` column exist.
- **Sub-workflow reference:** None.

#### Order Placed Ack.
- **Type and technical role:** `n8n-nodes-base.gmail`; sends an order confirmation email.
- **Configuration choices:**
  - Recipient: `{{ $json.email }}`
  - Subject: `Your Order Is Confirmed`
  - HTML email body references:
    - `{{ $json.client_name }}`
    - `{{ $json.product_name }}`
    - `{{ $json.estimated_delivery_date.split('T')[0] }}`
- **Key expressions or variables used:**
  - `$json.email`
  - `$json.client_name`
  - `$json.product_name`
  - `$json.estimated_delivery_date`
- **Input and output connections:** Input from **Execute a SQL query**; output to **Wait until product get deliver**.
- **Version-specific requirements:** `typeVersion: 2.2`.
- **Edge cases or potential failure types:**
  - Requires valid Gmail credentials.
  - If `estimated_delivery_date` is null, not a string, or does not contain `T`, the expression may fail.
  - If `email` is missing or malformed, Gmail send may fail.
  - HTML template assumes all customer/order fields are populated.
- **Sub-workflow reference:** None.

---

## 2.2 Delivery Wait & Delivery Status Recheck Loop

### Overview
This block delays processing until the expected delivery period has passed, then checks whether the order has actually been delivered. If not, it waits one more day and rechecks, creating a polling loop until `delivered = true`.

### Nodes Involved
- Wait until product get deliver
- Select rows from a table
- If
- Wait for a day

### Node Details

#### Wait until product get deliver
- **Type and technical role:** `n8n-nodes-base.wait`; pauses execution for a fixed duration.
- **Configuration choices:** Waits 7 days.
- **Key expressions or variables used:** None.
- **Input and output connections:** Input from **Order Placed Ack.**; output to **Select rows from a table**.
- **Version-specific requirements:** `typeVersion: 1.1`.
- **Edge cases or potential failure types:**
  - Requires n8n to persist waiting executions correctly.
  - If the estimated delivery timing varies by order, this fixed 7-day wait may not match actual expected delivery dates.
  - Long waits increase dependency on queue/execution persistence.
- **Sub-workflow reference:** None.

#### Select rows from a table
- **Type and technical role:** `n8n-nodes-base.postgres`; fetches the latest order state from the `orders` table.
- **Configuration choices:**
  - Operation: `select`
  - Schema: `public`
  - Table: `orders`
  - WHERE clause:
    - `order_id = {{ $('Execute a SQL query').item.json.order_id }}`
- **Key expressions or variables used:**
  - `$('Execute a SQL query').item.json.order_id`
- **Input and output connections:** Inputs from **Wait until product get deliver** and **Wait for a day**; output to **If**.
- **Version-specific requirements:** `typeVersion: 2.6`.
- **Edge cases or potential failure types:**
  - Depends on the earlier SQL node still being accessible by expression context.
  - If multiple rows match the same `order_id`, downstream logic may behave unpredictably.
  - If no row is returned, the IF node may not run as intended.
  - Requires `order_id` field in the upstream data and database table.
- **Sub-workflow reference:** None.

#### If
- **Type and technical role:** `n8n-nodes-base.if`; branches based on whether the order is delivered.
- **Configuration choices:**
  - Checks whether `{{ $json.delivered }}` equals boolean `true`.
  - Uses strict type validation and case-sensitive settings.
- **Key expressions or variables used:**
  - `$json.delivered`
- **Input and output connections:** Input from **Select rows from a table**.
  - **False output** goes nowhere.
  - **True output** goes to **Wait for a day**.
- **Version-specific requirements:** `typeVersion: 2.3`.
- **Edge cases or potential failure types:**
  - The connection logic appears reversed relative to the workflow description.
  - In n8n, output index 0 is usually “true” and output index 1 is “false”. Here, the node is connected as:
    - first output: empty
    - second output: **Wait for a day**
  - This means when `delivered` is **false**, the workflow waits another day; when `delivered` is **true**, it stops.
  - As currently wired, the workflow never proceeds to the AI blocks because there is no connection from this node to the AI processing nodes.
- **Sub-workflow reference:** None.

#### Wait for a day
- **Type and technical role:** `n8n-nodes-base.wait`; delays before rechecking order status.
- **Configuration choices:** Waits 1 day.
- **Key expressions or variables used:** None.
- **Input and output connections:** Input from **If** false branch; output back to **Select rows from a table**.
- **Version-specific requirements:** `typeVersion: 1.1`.
- **Edge cases or potential failure types:**
  - Long-running loop can continue indefinitely if `delivered` is never updated.
  - A missing timeout/escalation path means orders can stay in this loop forever.
- **Sub-workflow reference:** None.

### Important Implementation Observation
The AI follow-up branch is not connected from the delivery-check block. Based on the sticky notes and node naming, the intended behavior was likely:
- If `delivered = true` → continue to usage tips AI block
- If `delivered = false` → wait 1 day and recheck

But the current JSON only implements the recheck branch and does not connect delivery success to the next stage.

---

## 2.3 AI-Generated Product Usage Tips

### Overview
This block is intended to generate practical usage guidance for the purchased product using a Groq chat model through an AI Agent node. The generated text is transformed into an HTML list and emailed to the customer.

### Nodes Involved
- Get Product Usage Tips
- Groq Chat Model
- Format AI response
- Send Tips to User

### Node Details

#### Get Product Usage Tips
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent`; AI agent node intended to orchestrate an LLM request.
- **Configuration choices:** No parameters are configured in the JSON.
- **Key expressions or variables used:** None currently configured.
- **Input and output connections:** No incoming or outgoing connections in the actual workflow JSON.
- **Version-specific requirements:** `typeVersion: 3.1`.
- **Edge cases or potential failure types:**
  - As currently configured, this node is incomplete.
  - No prompt/instructions are defined.
  - No model is connected in the `ai_languageModel` connection layer.
  - Since it has no main input connection, it never executes.
- **Sub-workflow reference:** None.

#### Groq Chat Model
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatGroq`; chat language model provider for LangChain-compatible AI nodes.
- **Configuration choices:**
  - Model: `llama-3.3-70b-versatile`
- **Key expressions or variables used:** None.
- **Input and output connections:** No visible connection in the JSON.
- **Version-specific requirements:** `typeVersion: 1`.
- **Edge cases or potential failure types:**
  - Requires Groq API credentials.
  - Not connected to the AI Agent node, so it is operationally unused.
  - Model availability may vary by account/region/service changes.
- **Sub-workflow reference:** None.

#### Format AI response
- **Type and technical role:** `n8n-nodes-base.code`; converts AI bullet-style text into HTML `<ul><li>` markup.
- **Configuration choices:** JavaScript logic:
  - Reads `original.output`
  - Splits text on `"* "`
  - Trims and filters entries
  - Wraps them in a `<ul>` list
  - Preserves original fields and adds `formattedOutput`
- **Key expressions or variables used:**
  - `item.json.output`
  - `formattedOutput`
- **Input and output connections:** No incoming connection in the actual workflow JSON; output to **Send Tips to User**.
- **Version-specific requirements:** `typeVersion: 2`.
- **Edge cases or potential failure types:**
  - Assumes AI output uses `* ` bullet formatting.
  - If the AI returns paragraphs, numbered lists, markdown variants, or JSON, formatting may produce poor output.
  - HTML is inserted without escaping, so malicious or malformed model output could render unexpectedly.
- **Sub-workflow reference:** None.

#### Send Tips to User
- **Type and technical role:** `n8n-nodes-base.gmail`; sends the usage tips email.
- **Configuration choices:**
  - Recipient: `{{ $('Execute a SQL query').item.json.email }}`
  - Subject: `Getting Started with Your {{ $('Execute a SQL query').item.json.product_name }}`
  - HTML body uses:
    - `{{ $('Execute a SQL query').item.json.product_name }}`
    - `{{$json.formattedOutput}}`
  - The greeting hardcodes the name as `Devansh` instead of using dynamic order data.
- **Key expressions or variables used:**
  - `$('Execute a SQL query').item.json.email`
  - `$('Execute a SQL query').item.json.product_name`
  - `$json.formattedOutput`
- **Input and output connections:** Input from **Format AI response**; output to **Wait for 2 weeks**.
- **Version-specific requirements:** `typeVersion: 2.2`.
- **Edge cases or potential failure types:**
  - Requires Gmail credentials.
  - Hardcoded customer name may be incorrect for most recipients.
  - Depends on expression context from **Execute a SQL query** surviving across waits and disconnected branches.
  - If `formattedOutput` is missing, email content will be empty or malformed.
- **Sub-workflow reference:** None.

### Important Implementation Observation
This entire block is currently disconnected from the executed path. It will not run unless connected manually from the delivery-success branch and the AI Agent is properly configured with prompt and model linkage.

---

## 2.4 AI-Generated Complementary Product Recommendations

### Overview
This block is intended to send a delayed upsell email two weeks after the usage tips email. It uses another AI agent to create complementary product recommendations and formats the output into an HTML list.

### Nodes Involved
- Wait for 2 weeks
- Get Complementary Products
- Groq Chat Model1
- Code in JavaScript
- Send Tips to User1

### Node Details

#### Wait for 2 weeks
- **Type and technical role:** `n8n-nodes-base.wait`; delays before upsell communication.
- **Configuration choices:** Waits 14 days.
- **Key expressions or variables used:** None.
- **Input and output connections:** Input from **Send Tips to User**; no outgoing connection in the actual JSON.
- **Version-specific requirements:** `typeVersion: 1.1`.
- **Edge cases or potential failure types:**
  - Long execution persistence required.
  - This node is connected only from the tips email node, which itself is not on the active execution path.
- **Sub-workflow reference:** None.

#### Get Complementary Products
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent`; intended to generate upsell recommendations.
- **Configuration choices:** No parameters configured.
- **Key expressions or variables used:** None currently configured.
- **Input and output connections:** No incoming or outgoing connections.
- **Version-specific requirements:** `typeVersion: 3.1`.
- **Edge cases or potential failure types:**
  - Node is incomplete and disconnected.
  - No prompt, no tool configuration, no model linkage.
- **Sub-workflow reference:** None.

#### Groq Chat Model1
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatGroq`; model provider for the second AI agent.
- **Configuration choices:**
  - Model: `llama-3.3-70b-versatile`
- **Key expressions or variables used:** None.
- **Input and output connections:** No visible connection.
- **Version-specific requirements:** `typeVersion: 1`.
- **Edge cases or potential failure types:**
  - Requires Groq credentials.
  - Unused unless explicitly connected to the AI agent.
- **Sub-workflow reference:** None.

#### Code in JavaScript
- **Type and technical role:** `n8n-nodes-base.code`; formats recommendation bullets into HTML.
- **Configuration choices:** JavaScript:
  - Reads `original.output`
  - Splits on `* `
  - Builds `<ul>` with `<li>`
  - Preserves original fields and appends `formattedOutput`
- **Key expressions or variables used:**
  - `item.json.output`
  - `formattedOutput`
- **Input and output connections:** No incoming connection in the actual JSON; output to **Send Tips to User1**.
- **Version-specific requirements:** `typeVersion: 2`.
- **Edge cases or potential failure types:**
  - Same formatting fragility as the earlier code node.
  - Depends on AI output shape.
- **Sub-workflow reference:** None.

#### Send Tips to User1
- **Type and technical role:** `n8n-nodes-base.gmail`; sends the upsell/recommendation email.
- **Configuration choices:**
  - Recipient: `{{ $('Execute a SQL query').item.json.email }}`
  - Subject: `You Might Love This with Your {{ $('Execute a SQL query').item.json.product_name }}`
  - Message references:
    - `{{ $('Execute a SQL query').item.json.client_name }}`
    - `{{ $('Execute a SQL query').item.json.product_name }}`
    - `{{$json.formattedOutput}}`
- **Key expressions or variables used:**
  - `$('Execute a SQL query').item.json.email`
  - `$('Execute a SQL query').item.json.client_name`
  - `$('Execute a SQL query').item.json.product_name`
  - `$json.formattedOutput`
- **Input and output connections:** Input from **Code in JavaScript**; no outgoing connection.
- **Version-specific requirements:** `typeVersion: 2.2`.
- **Edge cases or potential failure types:**
  - Requires Gmail credentials.
  - Depends on upstream context from a node far earlier in the workflow.
  - Subject contains a trailing newline in the stored configuration.
- **Sub-workflow reference:** None.

### Important Implementation Observation
Like the usage tips block, this block is not fully connected. The intended sequence appears to be:
1. Wait 2 weeks
2. Generate recommendations with AI
3. Format output
4. Send email

But only the email formatting/output side is partially connected, and the AI nodes are not wired into execution.

---

## 2.5 Canvas Documentation Notes

### Overview
These nodes do not execute business logic. They document the workflow’s purpose, implementation phases, and setup instructions directly on the n8n canvas.

### Nodes Involved
- Sticky Note
- Step -1 Trigger & Order Detection
- Sticky Note1
- Sticky Note2
- Sticky Note3

### Node Details

#### Sticky Note
- **Type and technical role:** `n8n-nodes-base.stickyNote`; documentation only.
- **Configuration choices:** Large introductory note describing workflow purpose, steps, and setup checklist.
- **Key expressions or variables used:** None.
- **Input and output connections:** None.
- **Version-specific requirements:** `typeVersion: 1`.
- **Edge cases or potential failure types:** None operational.
- **Sub-workflow reference:** None.

#### Step -1 Trigger & Order Detection
- **Type and technical role:** `n8n-nodes-base.stickyNote`; documentation only.
- **Configuration choices:** Labels Step 1 area for order detection and confirmation email.
- **Input and output connections:** None.
- **Version-specific requirements:** `typeVersion: 1`.
- **Edge cases or potential failure types:** None operational.
- **Sub-workflow reference:** None.

#### Sticky Note1
- **Type and technical role:** `n8n-nodes-base.stickyNote`; documentation only.
- **Configuration choices:** Describes delivery status check block.
- **Input and output connections:** None.
- **Version-specific requirements:** `typeVersion: 1`.
- **Edge cases or potential failure types:** None operational.
- **Sub-workflow reference:** None.

#### Sticky Note2
- **Type and technical role:** `n8n-nodes-base.stickyNote`; documentation only.
- **Configuration choices:** Describes product usage tips AI block.
- **Input and output connections:** None.
- **Version-specific requirements:** `typeVersion: 1`.
- **Edge cases or potential failure types:** None operational.
- **Sub-workflow reference:** None.

#### Sticky Note3
- **Type and technical role:** `n8n-nodes-base.stickyNote`; documentation only.
- **Configuration choices:** Describes product recommendations AI block.
- **Input and output connections:** None.
- **Version-specific requirements:** `typeVersion: 1`.
- **Edge cases or potential failure types:** None operational.
- **Sub-workflow reference:** None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Schedule Trigger | n8n-nodes-base.scheduleTrigger | Polls for new orders every 2 minutes |  | Execute a SQL query | ## Step 1 - Order Detection & Confirmation Email<br>Detects new orders placed and sends an order confirmation email. |
| Execute a SQL query | n8n-nodes-base.postgres | Retrieves recent orders from Postgres | Schedule Trigger | Order Placed Ack. | ## Step 1 - Order Detection & Confirmation Email<br>Detects new orders placed and sends an order confirmation email. |
| Order Placed Ack. | n8n-nodes-base.gmail | Sends order confirmation email | Execute a SQL query | Wait until product get deliver | ## Step 1 - Order Detection & Confirmation Email<br>Detects new orders placed and sends an order confirmation email. |
| Wait until product get deliver | n8n-nodes-base.wait | Waits 7 days before first delivery check | Order Placed Ack. | Select rows from a table | ## Step 2 - Delivery Status Check<br>Waits for delivery and rechecks the order status until it is marked as delivered. |
| Select rows from a table | n8n-nodes-base.postgres | Reloads order status by order_id | Wait until product get deliver; Wait for a day | If | ## Step 2 - Delivery Status Check<br>Waits for delivery and rechecks the order status until it is marked as delivered. |
| If | n8n-nodes-base.if | Tests whether `delivered` is true | Select rows from a table | Wait for a day (false branch only) | ## Step 2 - Delivery Status Check<br>Waits for delivery and rechecks the order status until it is marked as delivered. |
| Wait for a day | n8n-nodes-base.wait | Delays 1 day before rechecking delivery | If | Select rows from a table | ## Step 2 - Delivery Status Check<br>Waits for delivery and rechecks the order status until it is marked as delivered. |
| Get Product Usage Tips | @n8n/n8n-nodes-langchain.agent | Intended AI agent for usage tips generation |  |  | ## Step 3 - Product Usage Tips (AI)<br>Generates product usage tips using AI and sends them to the customer. |
| Groq Chat Model | @n8n/n8n-nodes-langchain.lmChatGroq | Intended LLM backend for usage tips agent |  |  | ## Step 3 - Product Usage Tips (AI)<br>Generates product usage tips using AI and sends them to the customer. |
| Format AI response | n8n-nodes-base.code | Converts AI bullet text to HTML list |  | Send Tips to User | ## Step 3 - Product Usage Tips (AI)<br>Generates product usage tips using AI and sends them to the customer. |
| Send Tips to User | n8n-nodes-base.gmail | Sends product usage tips email | Format AI response | Wait for 2 weeks | ## Step 3 - Product Usage Tips (AI)<br>Generates product usage tips using AI and sends them to the customer. |
| Wait for 2 weeks | n8n-nodes-base.wait | Delays upsell follow-up by 14 days | Send Tips to User |  | ## Step 4 - Product Recommendations (AI)<br>Waits 2 weeks and sends AI-generated complementary product recommendations based on last purchase. |
| Get Complementary Products | @n8n/n8n-nodes-langchain.agent | Intended AI agent for complementary product recommendations |  |  | ## Step 4 - Product Recommendations (AI)<br>Waits 2 weeks and sends AI-generated complementary product recommendations based on last purchase. |
| Groq Chat Model1 | @n8n/n8n-nodes-langchain.lmChatGroq | Intended LLM backend for recommendation agent |  |  | ## Step 4 - Product Recommendations (AI)<br>Waits 2 weeks and sends AI-generated complementary product recommendations based on last purchase. |
| Code in JavaScript | n8n-nodes-base.code | Converts recommendation text to HTML list |  | Send Tips to User1 | ## Step 4 - Product Recommendations (AI)<br>Waits 2 weeks and sends AI-generated complementary product recommendations based on last purchase. |
| Send Tips to User1 | n8n-nodes-base.gmail | Sends complementary product recommendation email | Code in JavaScript |  | ## Step 4 - Product Recommendations (AI)<br>Waits 2 weeks and sends AI-generated complementary product recommendations based on last purchase. |
| Sticky Note | n8n-nodes-base.stickyNote | Canvas documentation for overall workflow |  |  | ## Order Placed → Delivery-Based Upsell Automation<br>This workflow automates post-purchase engagement by confirming new orders, tracking delivery status, sending helpful product usage tips, and following up with personalized upsell recommendations to increase repeat purchases.<br>### How it works<br>Step 1: A scheduled trigger checks for new orders and sends an order confirmation email immediately.<br>Step 2: The workflow waits until the expected delivery time, then checks if the order is delivered. If not, it waits 1 day and checks again. This repeats until delivery is confirmed.<br>Step 3: Once delivered, an AI agent generates short product usage tips. These are formatted and emailed to the customer.<br>Step 4: After 2 weeks, another AI agent generates complementary product suggestions, which are sent as an upsell email.<br>### Setup steps<br>1. Connect your Postgres database<br>2. Configure Gmail credentials<br>3. Ensure delivery status field is correct<br>4. Add Groq/OpenAI credentials<br>5. Review AI prompts if needed<br>6. Activate and test with sample data |
| Step -1 Trigger & Order Detection | n8n-nodes-base.stickyNote | Canvas note for Step 1 area |  |  | ## Step 1 - Order Detection & Confirmation Email<br>Detects new orders placed and sends an order confirmation email. |
| Sticky Note1 | n8n-nodes-base.stickyNote | Canvas note for Step 2 area |  |  | ## Step 2 - Delivery Status Check<br>Waits for delivery and rechecks the order status until it is marked as delivered. |
| Sticky Note2 | n8n-nodes-base.stickyNote | Canvas note for Step 3 area |  |  | ## Step 3 - Product Usage Tips (AI)<br>Generates product usage tips using AI and sends them to the customer. |
| Sticky Note3 | n8n-nodes-base.stickyNote | Canvas note for Step 4 area |  |  | ## Step 4 - Product Recommendations (AI)<br>Waits 2 weeks and sends AI-generated complementary product recommendations based on last purchase. |

---

# 4. Reproducing the Workflow from Scratch

Below is the practical build order to recreate the workflow in n8n. Because the provided JSON is partially disconnected, this section includes both:
- the workflow exactly as represented, and
- the intended missing connections/configuration needed to make the described automation actually work.

## 4.1 Prepare credentials and data model

1. **Create PostgreSQL credentials**
   - In n8n, add a Postgres credential.
   - Provide host, port, database, username, password, and SSL options as required.
   - Ensure access to a table named `orders`.

2. **Confirm expected database fields**
   Your `orders` table should at minimum provide:
   - `order_id`
   - `email`
   - `client_name`
   - `product_name`
   - `created_at`
   - `estimated_delivery_date`
   - `delivered`

3. **Create Gmail credentials**
   - Add Gmail OAuth2 credentials in n8n.
   - Grant permission to send email from the desired mailbox.

4. **Create Groq credentials**
   - Add Groq API credentials in n8n.
   - Confirm the selected model `llama-3.3-70b-versatile` is available.

---

## 4.2 Build Step 1: order detection and confirmation

1. **Add a Schedule Trigger node**
   - Name: `Schedule Trigger`
   - Type: `Schedule Trigger`
   - Set interval to:
     - Every `2` minutes

2. **Add a Postgres node**
   - Name: `Execute a SQL query`
   - Type: `Postgres`
   - Operation: `Execute Query`
   - SQL:
     ```sql
     SELECT *
     FROM orders
     WHERE created_at >= NOW() - INTERVAL '2 minute'
     ORDER BY created_at ASC;
     ```
   - Connect `Schedule Trigger -> Execute a SQL query`

3. **Add a Gmail node**
   - Name: `Order Placed Ack.`
   - Type: `Gmail`
   - Operation: Send Email
   - To: `={{ $json.email }}`
   - Subject: `Your Order Is Confirmed`
   - Message: use the provided HTML template, with expressions:
     - `{{ $json.client_name }}`
     - `{{ $json.product_name }}`
     - `{{ $json.estimated_delivery_date.split('T')[0] }}`
   - Connect `Execute a SQL query -> Order Placed Ack.`

---

## 4.3 Build Step 2: delivery waiting and recheck loop

1. **Add a Wait node**
   - Name: `Wait until product get deliver`
   - Type: `Wait`
   - Mode: After time interval
   - Amount: `7`
   - Unit: `Days`
   - Connect `Order Placed Ack. -> Wait until product get deliver`

2. **Add a Postgres node**
   - Name: `Select rows from a table`
   - Type: `Postgres`
   - Operation: `Select`
   - Schema: `public`
   - Table: `orders`
   - Add WHERE condition:
     - Column: `order_id`
     - Value: `={{ $('Execute a SQL query').item.json.order_id }}`
   - Connect `Wait until product get deliver -> Select rows from a table`

3. **Add an IF node**
   - Name: `If`
   - Type: `If`
   - Condition:
     - Left value: `={{ $json.delivered }}`
     - Operator: `equals`
     - Right value: `true`
     - Data type: Boolean
   - Connect `Select rows from a table -> If`

4. **Add another Wait node**
   - Name: `Wait for a day`
   - Type: `Wait`
   - Amount: `1`
   - Unit: `Days`

5. **Connect the recheck loop**
   - Connect the **false** output of `If` to `Wait for a day`
   - Connect `Wait for a day -> Select rows from a table`

### Important correction
To make the workflow function as described:
- **True branch** of `If` should continue to the AI usage-tips step.
- **False branch** should go to `Wait for a day`.

The JSON currently reflects only the false-branch loop and does not connect the true branch onward.

---

## 4.4 Build Step 3: product usage tips with AI

1. **Add a Groq Chat Model node**
   - Name: `Groq Chat Model`
   - Type: `Groq Chat Model`
   - Model: `llama-3.3-70b-versatile`

2. **Add an AI Agent node**
   - Name: `Get Product Usage Tips`
   - Type: `AI Agent`
   - Configure system/user instructions such as:
     - “Generate 3 to 5 short, practical usage tips for a customer who purchased {{ $json.product_name }}. Return the answer as bullet points starting with `* `.”
   - Connect the Groq model to this AI Agent through the AI model connection in the n8n UI.
   - Connect the **true** output of `If` to `Get Product Usage Tips`

3. **Add a Code node**
   - Name: `Format AI response`
   - Type: `Code`
   - Language: JavaScript
   - Paste logic equivalent to:
     - Read `item.json.output`
     - Split on `* `
     - Convert to `<ul><li>...</li></ul>`
     - Preserve all original fields
     - Save HTML in `formattedOutput`
   - Connect `Get Product Usage Tips -> Format AI response`

4. **Add a Gmail node**
   - Name: `Send Tips to User`
   - Type: `Gmail`
   - To: `={{ $('Execute a SQL query').item.json.email }}`
   - Subject: `=Getting Started with Your {{ $('Execute a SQL query').item.json.product_name }}`
   - Message: use the HTML template with:
     - Product name from `$('Execute a SQL query').item.json.product_name`
     - Inject `{{$json.formattedOutput}}`
   - Prefer replacing the hardcoded `Devansh` with:
     - `{{ $('Execute a SQL query').item.json.client_name }}`
   - Connect `Format AI response -> Send Tips to User`

---

## 4.5 Build Step 4: complementary product recommendations with AI

1. **Add a Wait node**
   - Name: `Wait for 2 weeks`
   - Type: `Wait`
   - Amount: `14`
   - Unit: `Days`
   - Connect `Send Tips to User -> Wait for 2 weeks`

2. **Add a second Groq Chat Model node**
   - Name: `Groq Chat Model1`
   - Type: `Groq Chat Model`
   - Model: `llama-3.3-70b-versatile`

3. **Add a second AI Agent node**
   - Name: `Get Complementary Products`
   - Type: `AI Agent`
   - Configure instructions such as:
     - “Based on the purchased product {{ $json.product_name }}, suggest 2 to 4 complementary products or accessories. Keep the suggestions concise and format each line as a bullet beginning with `* `.”
   - Connect `Groq Chat Model1` as the model for this AI Agent.
   - Connect `Wait for 2 weeks -> Get Complementary Products`

4. **Add a second Code node**
   - Name: `Code in JavaScript`
   - Type: `Code`
   - Language: JavaScript
   - Use the same list-formatting pattern as the first Code node.
   - Connect `Get Complementary Products -> Code in JavaScript`

5. **Add a final Gmail node**
   - Name: `Send Tips to User1`
   - Type: `Gmail`
   - To: `={{ $('Execute a SQL query').item.json.email }}`
   - Subject: `=You Might Love This with Your {{ $('Execute a SQL query').item.json.product_name }}`
   - Message: use the recommendation email HTML template, referencing:
     - `{{ $('Execute a SQL query').item.json.client_name }}`
     - `{{ $('Execute a SQL query').item.json.product_name }}`
     - `{{$json.formattedOutput}}`
   - Connect `Code in JavaScript -> Send Tips to User1`

---

## 4.6 Add documentation notes on the canvas

1. Add a sticky note for the overall workflow summary.
2. Add one sticky note for each block:
   - Step 1 - Order Detection & Confirmation Email
   - Step 2 - Delivery Status Check
   - Step 3 - Product Usage Tips (AI)
   - Step 4 - Product Recommendations (AI)

These notes are optional for execution but useful for maintenance.

---

## 4.7 Recommended fixes before production use

1. **Prevent duplicate order processing**
   - Add a `processed` or `confirmation_sent` flag in the database.
   - Update the SQL query to select only unprocessed orders.
   - Mark each order after confirmation email is sent.

2. **Use dynamic delivery timing**
   - Replace the fixed 7-day wait with logic based on `estimated_delivery_date`.

3. **Correct the IF branching**
   - True: proceed to AI tips
   - False: wait 1 day and retry

4. **Connect the AI nodes**
   - In the provided JSON, the AI blocks are disconnected and incomplete.

5. **Stabilize data references**
   - Instead of relying on `$('Execute a SQL query').item.json...` across long waits, preserve needed fields in the current item or re-fetch by `order_id`.

6. **Harden AI output handling**
   - Add fallback parsing if the model does not return bullet points.
   - Optionally sanitize HTML output.

7. **Replace hardcoded customer name**
   - In `Send Tips to User`, replace `Devansh` with a dynamic expression.

8. **Add failure monitoring**
   - Consider Error Trigger workflows, retry handling, or dead-letter notification.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Order Placed → Delivery-Based Upsell Automation | Overall workflow concept |
| This workflow automates post-purchase engagement by confirming new orders, tracking delivery status, sending helpful product usage tips, and following up with personalized upsell recommendations to increase repeat purchases. | Overall canvas note |
| Step 1: A scheduled trigger checks for new orders and sends an order confirmation email immediately. | Workflow logic summary |
| Step 2: The workflow waits until the expected delivery time, then checks if the order is delivered. If not, it waits 1 day and checks again. This repeats until delivery is confirmed. | Workflow logic summary |
| Step 3: Once delivered, an AI agent generates short product usage tips. These are formatted and emailed to the customer. | Workflow logic summary |
| Step 4: After 2 weeks, another AI agent generates complementary product suggestions, which are sent as an upsell email. | Workflow logic summary |
| Connect your Postgres database | Setup note |
| Configure Gmail credentials | Setup note |
| Ensure delivery status field is correct | Setup note |
| Add Groq/OpenAI credentials | Setup note |
| Review AI prompts if needed | Setup note |
| Activate and test with sample data | Setup note |

## Final assessment
The workflow idea is clear and commercially useful, but the exported JSON is only partially operational. The order confirmation and delivery recheck loop are implemented, while the AI follow-up sections are present as design intent but are not fully wired or configured. To make the workflow run end-to-end, the main required changes are:
- connect the `If` true branch to the first AI block,
- connect the AI nodes to their Groq models,
- add prompts to both AI agents,
- connect the second AI block after the 2-week wait,
- and improve duplicate handling plus data persistence across long waits.