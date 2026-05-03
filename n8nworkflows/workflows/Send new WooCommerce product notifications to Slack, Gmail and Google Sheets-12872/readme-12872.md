Send new WooCommerce product notifications to Slack, Gmail and Google Sheets

https://n8nworkflows.xyz/workflows/send-new-woocommerce-product-notifications-to-slack--gmail-and-google-sheets-12872


# Send new WooCommerce product notifications to Slack, Gmail and Google Sheets

## 1. Workflow Overview

**Workflow name:** Retail – New Product Drop Notifications  
**Purpose:** When a new product is created in WooCommerce, the workflow standardizes key product fields, then **notifies a team in Slack**, **sends a product launch email via Gmail**, and **logs the product to Google Sheets** for tracking.

### 1.1 Trigger & Input Reception
- Listens to WooCommerce’s `product.created` event and receives the product payload.

### 1.2 Data Preparation / Normalization
- Extracts and reshapes only the fields needed downstream (name, status, price, permalink, images) into a predictable structure.

### 1.3 Notifications
- Posts a formatted Slack message (including image URLs).
- Sends an HTML email announcement.

### 1.4 Logging
- Appends a row to a Google Sheet with product details and a timestamp.

---

## 2. Block-by-Block Analysis

### Block 1 — Trigger & Data Prep
**Overview:** Detects newly created products in WooCommerce and prepares a standardized product object reused by all downstream nodes.  
**Nodes Involved:**  
- WooCommerce – New Product Created  
- Format Product Details

#### Node: WooCommerce – New Product Created
- **Type / role:** `wooCommerceTrigger` — webhook-based trigger for WooCommerce events.
- **Configuration (interpreted):**
  - Event: **product.created**
  - Uses WooCommerce REST API credentials (“WooCommerce account 6”).
- **Key data expectations:**
  - Downstream nodes assume the trigger outputs a JSON structure containing **`body`** with fields like `name`, `status`, `price`, `permalink`, `images`.
- **Connections:**
  - **Output →** Format Product Details
- **Version notes:** TypeVersion **1**.
- **Edge cases / failures:**
  - Credential/auth failure (invalid consumer key/secret, wrong store URL).
  - WooCommerce not sending expected fields (custom setups may omit `price`, `images`, etc.).
  - Trigger/webhook delivery issues (site can’t reach n8n URL; firewall; SSL).
  - Payload shape mismatch (e.g., product object not under `body` depending on trigger implementation/version).

#### Node: Format Product Details
- **Type / role:** `set` — normalizes/whitelists fields for reuse.
- **Configuration (interpreted):**
  - Creates fields:
    - `body.name` = `{{$json.body.name}}`
    - `body.status` = `{{$json.body.status}}`
    - `body.price` = `{{$json.body.price}}`
    - `body.permalink` = `{{$json.body.permalink}}`
    - `body.images` = `{{$json.body.images}}` (array)
  - Keeps a consistent `body.*` structure for later expressions.
- **Connections:**
  - **Input ←** WooCommerce – New Product Created
  - **Output →** Notify Team on Slack
- **Version notes:** TypeVersion **3.4**
- **Edge cases / failures:**
  - If `$json.body.images` is missing/null, later nodes that call `.map(...)` can error unless guarded.
  - If price is not set (variable products, drafts), Slack/email/sheets may show blanks.

---

### Block 2 — Notifications
**Overview:** Announces the new product in Slack and via a formatted HTML email using the standardized fields from the Set node.  
**Nodes Involved:**  
- Notify Team on Slack  
- Send Product Launch Email

#### Node: Notify Team on Slack
- **Type / role:** `slack` — sends a message to a Slack channel.
- **Configuration (interpreted):**
  - Message text is built from expressions:
    - Name, Price, URL
    - Images list: `{{$json.body.images.map(i => i.src).join('\n')}}`
  - Channel selection is enabled, but **`channelId.value` is empty** in the JSON (must be set before production use).
- **Connections:**
  - **Input ←** Format Product Details
  - **Output →** Send Product Launch Email
- **Version notes:** TypeVersion **2.3**
- **Edge cases / failures:**
  - **Missing channel ID**: message will fail unless a default is chosen in UI or configured.
  - If `body.images` is undefined/non-array, `.map(...)` will throw (consider using optional chaining like `body.images?.map(...)`).
  - Slack auth/scopes missing (chat:write, channel access), or posting to a private channel without membership.

#### Node: Send Product Launch Email
- **Type / role:** `gmail` — sends an email through Gmail OAuth2.
- **Configuration (interpreted):**
  - Subject: `New Arrival: {{ $('Format Product Details').item.json.body.name }}`
  - HTML body: rich template with product name/price/status, link, and renders all images using:
    - `$('Format Product Details').item.json.body.images.map(...).join('')`
  - References **the Set node by name** (`$('Format Product Details')...`) rather than using the incoming `$json`.
- **Important template issue (bug):**
  - The “Status” row contains malformed HTML:
    - `<{{ $('Format Product Details').item.json.body.status }}/td>`
    - This should be something like: `<td>{{ ...status }}</td>`
  - As written, many email clients will render broken markup or drop parts of the table.
- **Connections:**
  - **Input ←** Notify Team on Slack
  - **Output →** Log Product in Google Sheets
- **Version notes:** TypeVersion **2.1**
- **Edge cases / failures:**
  - Gmail OAuth token expired/invalid; “restricted scopes” issues in some Google Workspace environments.
  - If images array is missing, `.map(...)` will throw (no optional chaining here).
  - Email size limits: many images/long URLs can push message size upward.
  - Using `$('Format Product Details').item` assumes that node executed and item indexing matches (usually fine in a linear flow, but can break if branching/merging is later added).

---

### Block 3 — Logging
**Overview:** Appends a row into a Google Sheet to keep a historical log of created products.  
**Nodes Involved:**  
- Log Product in Google Sheets

#### Node: Log Product in Google Sheets
- **Type / role:** `googleSheets` — appends a row to a sheet.
- **Configuration (interpreted):**
  - Operation: **Append**
  - Document: “(Retail) New Product Drop Notifications” (Spreadsheet ID: `16f1RrNM-e_AEUGHApNXnUDxGb_xPSxxZlvtcLowbeg0`)
  - Sheet/tab: `Sheet1` (gid=0)
  - Columns mapped (strings, no conversion):
    - Product Name = `Format Product Details.body.name`
    - Price = `...body.price`
    - Status = `...body.status`
    - Product URL = `...body.permalink`
    - Image URL = `...body.images?.map(i => i.src).join('\n')` (optional chaining used here—good)
    - Created At = `{{$now}}` (note: expression has a leading space in the JSON: `= {{ $now }}`)
  - Type conversion disabled (`attemptToConvertTypes: false`, `convertFieldsToString: false`)
- **Connections:**
  - **Input ←** Send Product Launch Email
  - **Output →** none (end of workflow)
- **Version notes:** TypeVersion **4.7**
- **Edge cases / failures:**
  - Spreadsheet permissions/auth issues (OAuth not authorized for that file).
  - Header mismatch: append mapping assumes these column headers exist exactly (“Product Name”, “Price”, etc.).
  - The `Created At` expression formatting may produce an ISO datetime; if you want Sheets-native date formatting, consider writing a formatted string or enabling type conversion.
  - API rate limits if many products are created in bursts.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| WooCommerce – New Product Created | wooCommerceTrigger | Entry trigger on new WooCommerce product creation | — | Format Product Details | ### Trigger & Data Preparation<br>Detects newly created products in WooCommerce using the WooCommerce Trigger and prepares a standardized product data object in the Set node for reuse across the workflow. |
| Format Product Details | set | Normalizes product fields into a reusable structure | WooCommerce – New Product Created | Notify Team on Slack | ### Trigger & Data Preparation<br>Detects newly created products in WooCommerce using the WooCommerce Trigger and prepares a standardized product data object in the Set node for reuse across the workflow. |
| Notify Team on Slack | slack | Sends Slack announcement with product details and image URLs | Format Product Details | Send Product Launch Email | ### Notifications<br>Sends Slack and email notifications to announce new product creations using the prepared product data. |
| Send Product Launch Email | gmail | Sends HTML product announcement email | Notify Team on Slack | Log Product in Google Sheets | ### Notifications<br>Sends Slack and email notifications to announce new product creations using the prepared product data. |
| Log Product in Google Sheets | googleSheets | Appends product log row to Google Sheets | Send Product Launch Email | — | ### Logging<br>Stores product details and creation timestamps in Google Sheets for tracking and reference. |
| Workflow Overview | stickyNote | Documentation / purpose note | — | — | ## Retail – New Product Drop Notifications<br>### Purpose<br>Automatically notify teams and log records when a new product is created in WooCommerce, ensuring consistent communication without manual effort.<br>### How It Works<br>The workflow uses the WooCommerce Trigger to detect newly created products. Product details such as name, price, status, permalink and images are prepared in a single Set node and reused to send Slack notifications, email alerts and log entries in Google Sheets.<br>### Setup Steps<br>1. Configure WooCommerce REST API credentials.<br>2. Connect Slack, Gmail and Google Sheets in n8n.<br>3. Review the **Format Product Details** node to adjust fields if needed.<br>4. Activate the workflow to start processing new products. |
| Trigger & Data Prep | stickyNote | Documentation / section label | — | — | ### Trigger & Data Preparation<br>Detects newly created products in WooCommerce using the WooCommerce Trigger and prepares a standardized product data object in the Set node for reuse across the workflow. |
| Notifications | stickyNote | Documentation / section label | — | — | ### Notifications<br>Sends Slack and email notifications to announce new product creations using the prepared product data. |
| Logging | stickyNote | Documentation / section label | — | — | ### Logging<br>Stores product details and creation timestamps in Google Sheets for tracking and reference. |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it: **Retail – New Product Drop Notifications**
   - (Optional) Set execution order to **v1** (Workflow Settings → Execution order).

2. **Add trigger: WooCommerce**
   - Add node: **WooCommerce Trigger**
   - Event: **Product Created** (`product.created`)
   - Credentials: create/select **WooCommerce API** credentials:
     - Store URL
     - Consumer Key / Consumer Secret
     - Ensure REST API access is enabled on WooCommerce.
   - Connect this node to the next step.

3. **Add data normalization: Set**
   - Add node: **Set**
   - Name: **Format Product Details**
   - Add fields (as expressions):
     - `body.name` → `{{$json.body.name}}`
     - `body.status` → `{{$json.body.status}}`
     - `body.price` → `{{$json.body.price}}`
     - `body.permalink` → `{{$json.body.permalink}}`
     - `body.images` → `{{$json.body.images}}`
   - Connect: **WooCommerce Trigger → Format Product Details**

4. **Add Slack notification**
   - Add node: **Slack**
   - Name: **Notify Team on Slack**
   - Operation: send message (post)
   - Select: **Channel**
   - **Set Channel ID** (pick the target channel in the UI; the provided JSON has it blank)
   - Message text (expression-enabled), for example:
     - `New Product Live!`
     - `Name: {{ $json.body.name }}`
     - `Price: {{ $json.body.price }}`
     - `URL: {{ $json.body.permalink }}`
     - `Images:\n{{ $json.body.images.map(i => i.src).join('\n') }}`
   - Credentials: connect Slack OAuth/token credential with permission to post.
   - Connect: **Format Product Details → Notify Team on Slack**

5. **Add Gmail email**
   - Add node: **Gmail**
   - Name: **Send Product Launch Email**
   - Operation: **Send**
   - Subject (expression):
     - `New Arrival: {{ $('Format Product Details').item.json.body.name }}`
   - Message body: paste your HTML template and ensure expressions are enabled.
   - **Fix the broken “Status” HTML row**:
     - Use: `<td>{{ $('Format Product Details').item.json.body.status }}</td>`
   - Credentials: Gmail OAuth2 credential (authorize the sending account).
   - Connect: **Notify Team on Slack → Send Product Launch Email**

6. **Add Google Sheets logging**
   - Add node: **Google Sheets**
   - Name: **Log Product in Google Sheets**
   - Operation: **Append**
   - Select the Spreadsheet (by ID) and Sheet tab (e.g., Sheet1)
   - Map columns (ensure your sheet has matching headers):
     - Product Name → `{{ $('Format Product Details').item.json.body.name }}`
     - Price → `{{ $('Format Product Details').item.json.body.price }}`
     - Status → `{{ $('Format Product Details').item.json.body.status }}`
     - Product URL → `{{ $('Format Product Details').item.json.body.permalink }}`
     - Image URL → `{{ $('Format Product Details').item.json.body.images?.map(i => i.src).join('\n') }}`
     - Created At → `{{ $now }}`
   - Credentials: Google Sheets OAuth2 (must have edit access to the file).
   - Connect: **Send Product Launch Email → Log Product in Google Sheets**

7. **Add notes (optional but matches the provided workflow)**
   - Add Sticky Notes:
     - “Workflow Overview” with the overview text.
     - “Trigger & Data Prep”, “Notifications”, “Logging” section notes.

8. **Test and activate**
   - Create a test product in WooCommerce.
   - Verify:
     - Slack receives message (and the channel is correct).
     - Gmail email renders correctly (especially the Status row).
     - Sheets row appends with correct columns.
   - Activate the workflow.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques. | Disclaimer provided in the request |
| Ensure Slack `channelId` is configured (currently blank in the workflow JSON). | Prevents Slack node failure |
| Fix malformed HTML in Gmail template “Status” cell (`<{{ status }}/td>`). | Prevents broken email rendering |