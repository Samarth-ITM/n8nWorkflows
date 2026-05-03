Send a daily Shopify order report with Streamline Connector and Gmail

https://n8nworkflows.xyz/workflows/send-a-daily-shopify-order-report-with-streamline-connector-and-gmail-14228


# Send a daily Shopify order report with Streamline Connector and Gmail

# 1. Workflow Overview

This workflow generates and sends a daily Shopify order report by retrieving orders through the Streamline Connector, filtering for orders created within the last 24 hours, converting those orders into an HTML email, and sending the report via Gmail.

It supports two entry modes:
- **Scheduled execution** every 24 hours
- **Manual execution** for testing or on-demand reporting

## 1.1 Triggering and Execution Start
The workflow starts either on a schedule or from a manual trigger. Both paths converge into the same data retrieval step.

## 1.2 Shopify Order Retrieval
The workflow uses the **Streamline Connector** node to fetch multiple Shopify orders through a custom integration credential tied to a Shop ID.

## 1.3 Order Normalization and Time Filtering
The returned order payload is split into individual items, then filtered to keep only orders whose `metadata.date` is within the last 24 hours.

## 1.4 Email Rendering and Delivery
Filtered orders are transformed into a styled HTML report using a Code node, then sent via Gmail with the subject **Daily Order Report**.

## 1.5 In-Canvas Guidance
Several Sticky Note nodes provide setup guidance, operational instructions, and a completion cue for users.

---

# 2. Block-by-Block Analysis

## Block 1 — Workflow Guidance and Setup Notes

### Overview
This block contains visual documentation embedded in the workflow canvas. It explains prerequisite credential setup, usage instructions, and the expected result.

### Nodes Involved
- **Sticky Note**
- **Sticky Note2**
- **Sticky Note1**

### Node Details

#### Sticky Note
- **Type and technical role:** `n8n-nodes-base.stickyNote`; non-executing documentation node
- **Configuration choices:** Displays a warning to create the Streamline credential first and to enter the Shop ID during credential setup
- **Key expressions or variables used:** None
- **Input and output connections:** None
- **Version-specific requirements:** Type version 1
- **Edge cases or potential failure types:** None at runtime; however, ignoring its instruction may cause authentication or configuration failure in the Streamline node
- **Sub-workflow reference:** None

#### Sticky Note2
- **Type and technical role:** `n8n-nodes-base.stickyNote`; non-executing documentation node
- **Configuration choices:** Provides a description of the workflow and usage steps:
  1. Connect credentials
  2. Change the recipient in Gmail
  3. Save and activate the workflow
  4. Run it; expected nightly execution at midnight
- **Key expressions or variables used:** None
- **Input and output connections:** None
- **Version-specific requirements:** Type version 1
- **Edge cases or potential failure types:** None at runtime; operationally, failing to set the Gmail recipient prevents useful delivery
- **Sub-workflow reference:** None

#### Sticky Note1
- **Type and technical role:** `n8n-nodes-base.stickyNote`; non-executing documentation node
- **Configuration choices:** Displays a success-oriented message prompting the user to check their email
- **Key expressions or variables used:** None
- **Input and output connections:** None
- **Version-specific requirements:** Type version 1
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** None

---

## Block 2 — Entry Points and Triggering

### Overview
This block defines how the workflow starts. It provides one automated trigger and one manual trigger, both feeding the same order retrieval node.

### Nodes Involved
- **every_24h**
- **daily_order_report**

### Node Details

#### every_24h
- **Type and technical role:** `n8n-nodes-base.scheduleTrigger`; automated workflow entry point
- **Configuration choices:** Configured with an interval rule using the default interval structure. The sticky note says it runs every night at midnight, but the actual JSON only shows a generic interval configuration and does not explicitly encode a midnight cron expression
- **Key expressions or variables used:** None
- **Input and output connections:** No input; outputs to **Streamline Connector1**
- **Version-specific requirements:** Type version 1.2
- **Edge cases or potential failure types:**
  - Misalignment between documented timing and actual trigger configuration
  - Timezone differences between instance settings and user expectations
  - Workflow must be activated for schedule execution
- **Sub-workflow reference:** None

#### daily_order_report
- **Type and technical role:** `n8n-nodes-base.manualTrigger`; manual workflow entry point
- **Configuration choices:** Default manual trigger for testing and ad hoc runs
- **Key expressions or variables used:** None
- **Input and output connections:** No input; outputs to **Streamline Connector1**
- **Version-specific requirements:** Type version 1
- **Edge cases or potential failure types:**
  - Only works when manually executed
  - Useful for validation when the schedule is not yet active
- **Sub-workflow reference:** None

---

## Block 3 — Shopify Order Retrieval via Streamline

### Overview
This block fetches Shopify orders through the Streamline Connector integration. It is the single upstream source for all downstream order processing.

### Nodes Involved
- **Streamline Connector1**

### Node Details

#### Streamline Connector1
- **Type and technical role:** `n8n-nodes-streamline.streamline`; external service connector used to query order data
- **Configuration choices:**
  - **Resource:** `orderActions`
  - **Operation:** `getMany`
  - **Request options:** left empty, so default retrieval behavior is used
- **Key expressions or variables used:** None in the node parameters shown
- **Input and output connections:** Receives input from **every_24h** and **daily_order_report**; outputs to **split_orders**
- **Version-specific requirements:** Type version 1; requires the Streamline community/custom node to be installed and available in the n8n instance
- **Edge cases or potential failure types:**
  - Missing or invalid Streamline credentials
  - Missing Shop ID in credential setup
  - API auth failures
  - Empty results from Shopify/Streamline
  - Schema differences in returned order structure
  - Rate limiting or upstream API outages
- **Sub-workflow reference:** None

---

## Block 4 — Order Splitting and Last-24-Hour Filtering

### Overview
This block converts the retrieved order array into individual n8n items and filters them by order date so that only recent orders are included in the report.

### Nodes Involved
- **split_orders**
- **last_24_hours**

### Node Details

#### split_orders
- **Type and technical role:** `n8n-nodes-base.splitOut`; splits an array field into separate n8n items
- **Configuration choices:**
  - **Field to split out:** `data`
  - No additional options configured
- **Key expressions or variables used:** Uses the `data` field from the incoming JSON payload
- **Input and output connections:** Receives input from **Streamline Connector1**; outputs to **last_24_hours**
- **Version-specific requirements:** Type version 1
- **Edge cases or potential failure types:**
  - Incoming payload may not contain `data`
  - `data` may not be an array
  - Empty `data` array will produce no downstream items
  - If Streamline changes its response schema, this node may silently break the flow logic
- **Sub-workflow reference:** None

#### last_24_hours
- **Type and technical role:** `n8n-nodes-base.filter`; conditional filtering node
- **Configuration choices:**
  - Uses a single condition with combinator `and`
  - Compares `{{$json.metadata.date}}` with `{{$now.minus(24, 'hours')}}`
  - Operator: date/time **after**
  - Strict type validation is enabled
- **Key expressions or variables used:**
  - `={{ $json.metadata.date }}`
  - `={{ $now.minus(24, 'hours') }}`
- **Input and output connections:** Receives input from **split_orders**; outputs matching items to **prepare_email**
- **Version-specific requirements:** Type version 2.2 with condition version 2 syntax
- **Edge cases or potential failure types:**
  - Missing `metadata`
  - Missing or invalid `metadata.date`
  - Date parsing mismatch due to non-ISO format
  - Timezone mismatches between source data and n8n runtime
  - If no orders match, downstream nodes may receive zero items and no email may be sent
- **Sub-workflow reference:** None

---

## Block 5 — HTML Report Construction

### Overview
This block takes the filtered orders and builds a single HTML email body. It also normalizes different possible input shapes before rendering.

### Nodes Involved
- **prepare_email**

### Node Details

#### prepare_email
- **Type and technical role:** `n8n-nodes-base.code`; JavaScript transformation node
- **Configuration choices:**
  - Uses custom JavaScript to:
    1. Normalize input into an array of orders
    2. Define inline CSS styles
    3. Format order dates
    4. Build one HTML block per order
    5. Return one item containing `{ html }`
- **Key expressions or variables used:**
  - Uses the `items` array provided by n8n
  - Reads fields from `order.metadata`, including:
    - `id`
    - `order_status`
    - `total_price`
    - `date`
    - `fulfillment_status`
    - `trackingUrls`
    - `order_status_url`
    - `product_image`
- **Input and output connections:** Receives filtered order items from **last_24_hours**; outputs one HTML payload item to **send_email**
- **Version-specific requirements:** Type version 2
- **Edge cases or potential failure types:**
  - If the node receives zero items, `items[0]` access can fail depending on execution behavior
  - `trackingUrls` may be an array or object but is injected directly into HTML as a string
  - `order_status_url` may be undefined, producing a broken anchor tag
  - HTML rendering may contain `undefined` values if metadata is incomplete
  - Unescaped values could produce malformed HTML if upstream content contains special characters
  - Locale-sensitive `toLocaleString()` output may vary by server environment
- **Sub-workflow reference:** None

---

## Block 6 — Email Delivery

### Overview
This block sends the generated HTML report using Gmail. It is the final delivery step of the workflow.

### Nodes Involved
- **send_email**

### Node Details

#### send_email
- **Type and technical role:** `n8n-nodes-base.gmail`; email sending node using Gmail integration
- **Configuration choices:**
  - **Subject:** `Daily Order Report`
  - **Message:** `={{ $json.html }}`
  - **Append attribution:** disabled
  - The recipient address is expected to be configured manually in the node UI, as indicated by the sticky note, but it is not visible in the exported JSON snippet
- **Key expressions or variables used:**
  - `={{ $json.html }}`
- **Input and output connections:** Receives input from **prepare_email**; no downstream output connected
- **Version-specific requirements:** Type version 2.1; requires Gmail credentials configured in n8n
- **Edge cases or potential failure types:**
  - Missing Gmail credentials
  - OAuth permission issues
  - Missing recipient configuration
  - Gmail sending limits or quota restrictions
  - HTML may be sent as plain content depending on node configuration and Gmail node behavior
  - If upstream produces no item, this node will not execute
- **Sub-workflow reference:** None

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note | Sticky Note | Canvas documentation for Streamline credential setup |  |  | # ⚠️ Create your Streamline credential first |
| Sticky Note | Sticky Note | Canvas documentation for Streamline credential setup |  |  | Enter your Shop ID when creating the credential. Once saved, select that credential from the dropdown — just like any standard n8n node. You can delete this sticky once complete. |
| Sticky Note2 | Sticky Note | Canvas documentation for workflow purpose and usage |  |  | ## 🧩 Email Daily Shopify Order Report - Streamline Connector |
| Sticky Note2 | Sticky Note | Canvas documentation for workflow purpose and usage |  |  | ### A simple n8n workflow that fetches all orders from your Shopify store using Streamline Connector, then prepares and sends a daily order report. |
| Sticky Note2 | Sticky Note | Canvas documentation for workflow purpose and usage |  |  | ## ⚙️ How to Use |
| Sticky Note2 | Sticky Note | Canvas documentation for workflow purpose and usage |  |  | 1. Connect your credentials. |
| Sticky Note2 | Sticky Note | Canvas documentation for workflow purpose and usage |  |  | 2. Change the "to" in the Gmail node to your email. |
| Sticky Note2 | Sticky Note | Canvas documentation for workflow purpose and usage |  |  | 3. Save and turn on your workflow. |
| Sticky Note2 | Sticky Note | Canvas documentation for workflow purpose and usage |  |  | 4. Click run! This workflow will run every night at midnight. |
| split_orders | Split Out | Splits the returned order array into one item per order | Streamline Connector1 | last_24_hours | ## 🧩 Email Daily Shopify Order Report - Streamline Connector |
| split_orders | Split Out | Splits the returned order array into one item per order | Streamline Connector1 | last_24_hours | ### A simple n8n workflow that fetches all orders from your Shopify store using Streamline Connector, then prepares and sends a daily order report. |
| split_orders | Split Out | Splits the returned order array into one item per order | Streamline Connector1 | last_24_hours | ## ⚙️ How to Use |
| split_orders | Split Out | Splits the returned order array into one item per order | Streamline Connector1 | last_24_hours | 1. Connect your credentials. |
| split_orders | Split Out | Splits the returned order array into one item per order | Streamline Connector1 | last_24_hours | 2. Change the "to" in the Gmail node to your email. |
| split_orders | Split Out | Splits the returned order array into one item per order | Streamline Connector1 | last_24_hours | 3. Save and turn on your workflow. |
| split_orders | Split Out | Splits the returned order array into one item per order | Streamline Connector1 | last_24_hours | 4. Click run! This workflow will run every night at midnight. |
| last_24_hours | Filter | Keeps only orders newer than 24 hours | split_orders | prepare_email | ## 🧩 Email Daily Shopify Order Report - Streamline Connector |
| last_24_hours | Filter | Keeps only orders newer than 24 hours | split_orders | prepare_email | ### A simple n8n workflow that fetches all orders from your Shopify store using Streamline Connector, then prepares and sends a daily order report. |
| last_24_hours | Filter | Keeps only orders newer than 24 hours | split_orders | prepare_email | ## ⚙️ How to Use |
| last_24_hours | Filter | Keeps only orders newer than 24 hours | split_orders | prepare_email | 1. Connect your credentials. |
| last_24_hours | Filter | Keeps only orders newer than 24 hours | split_orders | prepare_email | 2. Change the "to" in the Gmail node to your email. |
| last_24_hours | Filter | Keeps only orders newer than 24 hours | split_orders | prepare_email | 3. Save and turn on your workflow. |
| last_24_hours | Filter | Keeps only orders newer than 24 hours | split_orders | prepare_email | 4. Click run! This workflow will run every night at midnight. |
| prepare_email | Code | Builds one HTML email body from filtered orders | last_24_hours | send_email | ## 🧩 Email Daily Shopify Order Report - Streamline Connector |
| prepare_email | Code | Builds one HTML email body from filtered orders | last_24_hours | send_email | ### A simple n8n workflow that fetches all orders from your Shopify store using Streamline Connector, then prepares and sends a daily order report. |
| prepare_email | Code | Builds one HTML email body from filtered orders | last_24_hours | send_email | ## ⚙️ How to Use |
| prepare_email | Code | Builds one HTML email body from filtered orders | last_24_hours | send_email | 1. Connect your credentials. |
| prepare_email | Code | Builds one HTML email body from filtered orders | last_24_hours | send_email | 2. Change the "to" in the Gmail node to your email. |
| prepare_email | Code | Builds one HTML email body from filtered orders | last_24_hours | send_email | 3. Save and turn on your workflow. |
| prepare_email | Code | Builds one HTML email body from filtered orders | last_24_hours | send_email | 4. Click run! This workflow will run every night at midnight. |
| send_email | Gmail | Sends the HTML report by email | prepare_email |  | ## 🧩 Email Daily Shopify Order Report - Streamline Connector |
| send_email | Gmail | Sends the HTML report by email | prepare_email |  | ### A simple n8n workflow that fetches all orders from your Shopify store using Streamline Connector, then prepares and sends a daily order report. |
| send_email | Gmail | Sends the HTML report by email | prepare_email |  | ## ⚙️ How to Use |
| send_email | Gmail | Sends the HTML report by email | prepare_email |  | 1. Connect your credentials. |
| send_email | Gmail | Sends the HTML report by email | prepare_email |  | 2. Change the "to" in the Gmail node to your email. |
| send_email | Gmail | Sends the HTML report by email | prepare_email |  | 3. Save and turn on your workflow. |
| send_email | Gmail | Sends the HTML report by email | prepare_email |  | 4. Click run! This workflow will run every night at midnight. |
| send_email | Gmail | Sends the HTML report by email | prepare_email |  | ## 🚀 Check your email for report! |
| Sticky Note1 | Sticky Note | Canvas documentation for expected result |  |  | ## 🚀 Check your email for report! |
| every_24h | Schedule Trigger | Automated workflow start |  | Streamline Connector1 |  |
| daily_order_report | Manual Trigger | Manual workflow start for testing |  | Streamline Connector1 |  |
| Streamline Connector1 | Streamline Connector | Retrieves Shopify orders via Streamline | every_24h, daily_order_report | split_orders | ## 🧩 Email Daily Shopify Order Report - Streamline Connector |
| Streamline Connector1 | Streamline Connector | Retrieves Shopify orders via Streamline | every_24h, daily_order_report | split_orders | ### A simple n8n workflow that fetches all orders from your Shopify store using Streamline Connector, then prepares and sends a daily order report. |
| Streamline Connector1 | Streamline Connector | Retrieves Shopify orders via Streamline | every_24h, daily_order_report | split_orders | ## ⚙️ How to Use |
| Streamline Connector1 | Streamline Connector | Retrieves Shopify orders via Streamline | every_24h, daily_order_report | split_orders | 1. Connect your credentials. |
| Streamline Connector1 | Streamline Connector | Retrieves Shopify orders via Streamline | every_24h, daily_order_report | split_orders | 2. Change the "to" in the Gmail node to your email. |
| Streamline Connector1 | Streamline Connector | Retrieves Shopify orders via Streamline | every_24h, daily_order_report | split_orders | 3. Save and turn on your workflow. |
| Streamline Connector1 | Streamline Connector | Retrieves Shopify orders via Streamline | every_24h, daily_order_report | split_orders | 4. Click run! This workflow will run every night at midnight. |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Open n8n and create a blank workflow.
   - Name it something like: **Send a daily Shopify order report with Streamline Connector and Gmail**.

2. **Add the first documentation note**
   - Create a **Sticky Note** node.
   - Add this content:
     - `# ⚠️ Create your Streamline credential first`
     - `Enter your Shop ID when creating the credential. Once saved, select that credential from the dropdown — just like any standard n8n node. You can delete this sticky once complete.`

3. **Add the second documentation note**
   - Create another **Sticky Note** node.
   - Add this content:
     - `## 🧩 Email Daily Shopify Order Report - Streamline Connector`
     - `### A simple n8n workflow that fetches all orders from your Shopify store using Streamline Connector, then prepares and sends a daily order report.`
     - `## ⚙️ How to Use`
     - `1. Connect your credentials.`
     - `2. Change the "to" in the Gmail node to your email.`
     - `3. Save and turn on your workflow.`
     - `4. Click run! This workflow will run every night at midnight.`
   - Optionally set its color to match the original informational note style.

4. **Add a Schedule Trigger node**
   - Create a **Schedule Trigger** node named **every_24h**.
   - Configure it with an interval-based schedule.
   - Important: the exported configuration does not explicitly show a midnight cron schedule, so if you want true midnight execution, configure the trigger accordingly in your own instance and verify timezone settings.

5. **Add a Manual Trigger node**
   - Create a **Manual Trigger** node named **daily_order_report**.
   - Leave default settings.
   - This will be your test entry point.

6. **Install or enable the Streamline Connector node**
   - Ensure your n8n instance has access to the **`n8n-nodes-streamline.streamline`** node.
   - If this is a community/custom node, install it before continuing.

7. **Create Streamline credentials**
   - In n8n credentials, create the credential required by the Streamline Connector.
   - Enter your **Shop ID** during credential creation.
   - Save the credential.
   - This is mandatory for the workflow to reach Shopify order data.

8. **Add the Streamline Connector node**
   - Create a node named **Streamline Connector1** using the Streamline node.
   - Set:
     - **Resource:** `orderActions`
     - **Operation:** `getMany`
     - **Request Options:** leave empty/default
   - Select the Streamline credential created earlier.

9. **Connect both triggers to the Streamline node**
   - Connect **every_24h → Streamline Connector1**
   - Connect **daily_order_report → Streamline Connector1**

10. **Add a Split Out node**
    - Create a **Split Out** node named **split_orders**.
    - Set **Field to Split Out** to `data`.
    - Leave additional options at default.

11. **Connect Streamline to Split Out**
    - Connect **Streamline Connector1 → split_orders**

12. **Add a Filter node**
    - Create a **Filter** node named **last_24_hours**.
    - Add one condition:
      - Left value: `={{ $json.metadata.date }}`
      - Operator: **Date & Time → after**
      - Right value: `={{ $now.minus(24, 'hours') }}`
    - Keep strict validation enabled if available.
    - Use `and` as the combinator, even though only one condition is present.

13. **Connect Split Out to Filter**
    - Connect **split_orders → last_24_hours**

14. **Add a Code node**
    - Create a **Code** node named **prepare_email**.
    - Set it to JavaScript mode.
    - Paste logic equivalent to the following behavior:
      - Normalize incoming items into an array of orders
      - Build inline HTML/CSS
      - Render each order using fields from `metadata`
      - Return a single item with `json.html`
   - Use these metadata fields in the renderer:
     - `id`
     - `order_status`
     - `total_price`
     - `date`
     - `fulfillment_status`
     - `trackingUrls`
     - `order_status_url`
     - `product_image`

15. **Use the email-building logic**
   - The node should:
     - Detect whether orders arrive as multiple items or a single array
     - Format dates via JavaScript `Date`
     - Build a `Daily Order Report` HTML document
     - Return:
       - `json: { html: "<html>...</html>" }`

16. **Connect Filter to Code**
    - Connect **last_24_hours → prepare_email**

17. **Create Gmail credentials**
    - In n8n credentials, create or connect a **Gmail** credential.
    - Authenticate with the Google account that will send the report.
    - Ensure the account has permission to send mail and that OAuth scopes are accepted.

18. **Add a Gmail node**
    - Create a **Gmail** node named **send_email**.
    - Set the message/body field to:
      - `={{ $json.html }}`
    - Set the subject to:
      - `Daily Order Report`
    - In options, disable:
      - **Append Attribution**
    - Set the recipient address in the **To** field to your target email address.
      - The exported JSON does not expose the recipient value, so this must be filled manually.

19. **Connect Code to Gmail**
    - Connect **prepare_email → send_email**

20. **Add the final documentation note**
    - Create a **Sticky Note** node.
    - Add:
      - `## 🚀 Check your email for report!`

21. **Validate response structure from Streamline**
    - Run the workflow manually from **daily_order_report**.
    - Confirm that:
      - Streamline returns a `data` array
      - Each order contains `metadata.date`
      - The expected metadata fields exist
    - If the structure differs, adjust the Split Out field or Code node accordingly.

22. **Validate Gmail delivery**
    - Run the workflow manually.
    - Confirm the email arrives and renders HTML correctly.
    - If the email body appears incorrectly, verify whether the Gmail node is sending the field as HTML content in your n8n version.

23. **Activate the workflow**
    - Save the workflow.
    - Activate it so the **Schedule Trigger** can run automatically.

24. **Recommended hardening improvements**
    - Add a fallback path when no orders are found in the last 24 hours
   - Optionally send a “no new orders” email instead of sending nothing
    - Add error handling for:
      - Streamline auth failures
      - Empty or malformed API payloads
      - Gmail send failures
    - Consider escaping HTML output and formatting tracking URLs more safely

### Sub-workflow setup
This workflow does **not** invoke any sub-workflows and does not require any child workflow configuration.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Create your Streamline credential first and enter your Shop ID during credential setup. | Applies to the Streamline Connector credential configuration |
| Change the Gmail `To` field to your own email address before production use. | Applies to the Gmail node |
| The canvas note says the workflow runs every night at midnight, but the exported Schedule Trigger configuration does not explicitly prove a midnight schedule. Verify the schedule manually in your instance. | Applies to the Schedule Trigger configuration |
| After setup, save and activate the workflow so the scheduled trigger can run automatically. | Operational note |
| Check your email after execution to confirm report delivery and HTML rendering. | Output verification |