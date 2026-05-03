Send weekly low‑stock alerts from WooCommerce to Slack and Jira

https://n8nworkflows.xyz/workflows/send-weekly-low-stock-alerts-from-woocommerce-to-slack-and-jira-13276


# Send weekly low‑stock alerts from WooCommerce to Slack and Jira

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:** This workflow monitors WooCommerce product inventory on a weekly schedule, detects items that have fallen below category-specific “low” and “urgent/very low” thresholds, then:
- Sends a **low stock** digest to a Slack channel.
- For **urgent low stock**, creates a **Jira issue** and posts a Slack alert including the Jira ticket details.

**Target use cases:**
- E-commerce operations teams needing proactive restock notifications.
- Engineering/ops teams that want urgent shortages tracked via Jira.

### 1.1 Scheduling & Orchestration
Runs automatically every Monday at midnight (as stated in notes; actual node config triggers weekly on Monday).

### 1.2 Data Retrieval (WooCommerce)
Fetches all products from WooCommerce.

### 1.3 Inventory Classification (Threshold Logic)
Filters to stock-managed products and classifies them into:
- `low_stock`
- `uregent_low_stock` (typo in key name is part of the workflow)

### 1.4 Low Stock Slack Notification
Builds a Slack Block Kit message listing low-stock products and posts it to a Slack channel.

### 1.5 Urgent Low Stock Jira + Slack Notification
Builds a Jira issue description, creates a Jira issue, then formats and posts an urgent Slack message including Jira `id` and `key`.

---

## 2. Block-by-Block Analysis

### Block 1 — Scheduling & Start
**Overview:** Starts the workflow automatically on a weekly schedule (Monday).  
**Nodes involved:** `Inventory Check Scheduler`

#### Node: Inventory Check Scheduler
- **Type / role:** Schedule Trigger — entry point.
- **Configuration (interpreted):**
  - Runs on a weekly interval.
  - `triggerAtDay: [1]` → Monday (n8n uses 1–7 depending on locale/version; here it’s intended as Monday per sticky notes).
- **Inputs/Outputs:**
  - **Input:** None (trigger node).
  - **Output:** Triggers `Get many products from WooCommerce`.
- **Version notes:** `typeVersion: 1.2`.
- **Potential failures / edge cases:**
  - Timezone differences: “midnight” depends on n8n instance timezone settings.
  - Misinterpretation of weekday indexing if instance locale differs (verify it triggers on the intended day).

---

### Block 2 — Fetch Products & Classify Inventory
**Overview:** Retrieves all products from WooCommerce and separates them into low/urgent low stock arrays using category-based thresholds.  
**Nodes involved:** `Get many products from WooCommerce`, `Separate products to low stock and very low stock`

#### Node: Get many products from WooCommerce
- **Type / role:** WooCommerce node — data extraction.
- **Configuration (interpreted):**
  - Operation: **Get All** products.
  - No extra options configured.
- **Credentials:** `WooCommerce account 5` (WooCommerce API).
- **Inputs/Outputs:**
  - **Input:** Trigger from scheduler.
  - **Output:** List of WooCommerce product items to the classification code node.
- **Version notes:** `typeVersion: 1`.
- **Potential failures / edge cases:**
  - Authentication/permission errors (invalid consumer key/secret).
  - Pagination/large catalogs: “getAll” can be slow or hit rate limits; may require limiting fields or paging depending on store size.
  - WooCommerce API timeouts for large stores.

#### Node: Separate products to low stock and very low stock
- **Type / role:** Code node — business rules & categorization.
- **Configuration (interpreted):**
  - Reads **all incoming WooCommerce product items** via `$input.all()`.
  - Skips products where:
    - `manage_stock` is false, or
    - `stock_quantity` is `null`
  - Determines a single category slug if **exactly one category** exists; otherwise uses `"default"`.
  - Compares stock quantity against two threshold maps:
    - `thresholds` (low stock)
    - `urgentThresholds` (very low stock)
  - Builds normalized objects:
    - `product` (name)
    - `current_stock` (stock_quantity)
    - `Category` (comma-joined category names)
  - Outputs one item with:
    - `json.low_stock` (array)
    - `json.uregent_low_stock` (array; note spelling)
- **Key variables/expressions:**
  - Uses `categories[0].slug.toLowerCase()` only if `categories.length === 1`.
  - Category thresholds exist for: `headphones`, `gaming-laptops`, `laptops`, `fashion`, `mobile-phones`, `monitors`, else `default`.
- **Inputs/Outputs:**
  - **Input:** Products from WooCommerce.
  - **Output:** Branches to:
    - `Format Low Stock Message`
    - `Format Urgent Low Stock Message for Jira`
- **Version notes:** `typeVersion: 2`.
- **Potential failures / edge cases:**
  - If a product has multiple categories, it’s treated as `"default"` (may under/over-alert).
  - Variable product stock handling: parent products may have `stock_quantity: null` while variations carry stock.
  - Key typo `uregent_low_stock` must remain consistent downstream; renaming requires updating multiple nodes.

---

### Block 3 — Low Stock Slack Notification
**Overview:** Formats a Slack Block Kit message for low-stock items and sends it to a Slack channel.  
**Nodes involved:** `Format Low Stock Message`, `Send Low Stock Alert to Slack`

#### Node: Format Low Stock Message
- **Type / role:** Code node — Slack Block Kit message builder.
- **Configuration (interpreted):**
  - Reads `low_stock` from the first input item: `$input.first().json.low_stock`.
  - If no low stock items, returns `[]` which **stops** downstream Slack message (no alert).
  - Produces an output shaped as a Slack message payload:
    - top-level `text`
    - `blocks` containing a `rich_text` block with per-product sections
  - For stock `null`, prints “Out of Stock”; otherwise prints numeric stock.
- **Key variables/expressions:**
  - `item.Category` (capital C) must exist from previous node.
- **Inputs/Outputs:**
  - **Input:** From `Separate products...`
  - **Output:** To `Send Low Stock Alert to Slack`
- **Version notes:** `typeVersion: 2`.
- **Potential failures / edge cases:**
  - Slack “rich_text” blocks require compatible Slack APIs; malformed block payloads will fail posting.
  - Large product lists may exceed Slack block limits (message size / block count).

#### Node: Send Low Stock Alert to Slack
- **Type / role:** Slack node — message delivery.
- **Configuration (interpreted):**
  - Posts to a specific channel: `daily_low_stock_update` (cached name).
  - Message type: **Block**.
  - Uses `blocksUi = {{ $json }}` (the Code node outputs the Slack payload).
  - `retryOnFail: true`.
- **Credentials:** `Slack account 32`.
- **Inputs/Outputs:**
  - **Input:** Slack payload from `Format Low Stock Message`.
  - **Output:** None used further.
- **Version notes:** `typeVersion: 2.3`.
- **Potential failures / edge cases:**
  - Slack auth scopes missing (chat:write, etc.).
  - Posting to a channel the bot isn’t a member of.
  - Payload shape mismatch (expects blocks object in the field used by node).

---

### Block 4 — Urgent Low Stock → Jira Issue
**Overview:** Builds a Jira issue description for urgent items and creates a high-priority issue.  
**Nodes involved:** `Format Urgent Low Stock Message for Jira`, `Create an Issue in Jira`

#### Node: Format Urgent Low Stock Message for Jira
- **Type / role:** Code node — Jira description generator.
- **Configuration (interpreted):**
  - Reads urgent array from `$input.first().json.uregent_low_stock`.
  - Creates a plain-text description listing products and stock counts.
  - Outputs: `{ description: message }`
- **Inputs/Outputs:**
  - **Input:** From `Separate products...`
  - **Output:** To `Create an Issue in Jira`
- **Version notes:** `typeVersion: 2`.
- **Potential failures / edge cases:**
  - If `uregent_low_stock` is empty/undefined, the code still creates a message header and could create a Jira ticket with an empty list (no guard clause here). If you want to avoid empty Jira issues, add an early return `[]`.

#### Node: Create an Issue in Jira
- **Type / role:** Jira Software Cloud node — issue creation.
- **Configuration (interpreted):**
  - Project: **My Software Team** (ID `10000`)
  - Issue type: **Bug** (ID `10007`)
  - Summary: `Urgent Low Stock - {{ $now.format('yyyy-MM-dd') }}`
  - Additional fields:
    - Assignee: `Fullstacktech wli`
    - Priority: **High**
    - Description: from the previous node: `{{ $json.description }}`
- **Credentials:** `Jira SW Cloud account 6`.
- **Inputs/Outputs:**
  - **Input:** Jira description object.
  - **Output:** Jira issue result (includes at least `id`, `key`) to `Format Urgent Low Stock Message`.
- **Version notes:** `typeVersion: 1`.
- **Potential failures / edge cases:**
  - Jira permissions: cannot assign issues, create issue, or set priority.
  - Field configuration mismatches (project may not allow “Bug”, priority, or assignee).
  - Rate limiting / intermittent Jira API failures.

---

### Block 5 — Urgent Low Stock → Slack with Jira Details
**Overview:** Formats a Slack urgent alert including Jira ticket identifiers, then posts to a dedicated urgent Slack channel.  
**Nodes involved:** `Format Urgent Low Stock Message`, `Send Urgent Low Stock Alert to Slack`

#### Node: Format Urgent Low Stock Message
- **Type / role:** Code node — Slack Block Kit message builder (urgent variant).
- **Configuration (interpreted):**
  - Fetches urgent list from the earlier classifier node using **node reference**:
    - `$('Separate products to low stock and very low stock').first().json.uregent_low_stock`
  - If no urgent items, returns `[]` (prevents Slack post).
  - Also expects Jira creation output as current `$input.first().json`, using:
    - `$input.first().json.id`
    - `$input.first().json.key`
  - Produces Slack payload with a message and appends Jira ticket id/key.
- **Inputs/Outputs:**
  - **Input:** From `Create an Issue in Jira` (Jira issue data).
  - **Output:** To `Send Urgent Low Stock Alert to Slack`.
- **Version notes:** `typeVersion: 2`.
- **Potential failures / edge cases:**
  - If Jira issue creation fails, this node won’t run; no Slack urgent alert will be sent.
  - If Jira node output doesn’t contain `id`/`key` as expected, Slack message will have missing fields.
  - Uses both `$input` and a cross-node reference `$()`; if the workflow is changed (node rename), the reference breaks.

#### Node: Send Urgent Low Stock Alert to Slack
- **Type / role:** Slack node — message delivery (urgent channel).
- **Configuration (interpreted):**
  - Channel: `daily_urgent_low_stock_update`
  - Message type: **Block**
  - `blocksUi = {{ $json }}`
  - `retryOnFail: true`
- **Credentials:** `Slack account 32`.
- **Inputs/Outputs:**
  - **Input:** Slack payload from urgent formatter.
  - **Output:** None used further.
- **Version notes:** `typeVersion: 2.3`.
- **Potential failures / edge cases:**
  - Same Slack considerations as low-stock channel (scopes, membership, block limits).
  - If message is too large due to many urgent items, Slack rejects it.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Inventory Check Scheduler | Schedule Trigger | Weekly entry point (Monday) | — | Get many products from WooCommerce | Runs the workflow automatically every Monday at midnight without manual intervention. |
| Get many products from WooCommerce | WooCommerce | Fetch all products | Inventory Check Scheduler | Separate products to low stock and very low stock | This part fetches all products from WooCommerce and analyzes their inventory levels. / Only products with managed stock are considered. / Each product’s stock quantity is compared against category-wise thresholds to determine whether it falls under: - Low stock - Very low (urgent) stock / The products are then separated into two arrays for further processing. |
| Separate products to low stock and very low stock | Code | Classify products into low vs urgent low stock | Get many products from WooCommerce | Format Low Stock Message; Format Urgent Low Stock Message for Jira | This part fetches all products from WooCommerce and analyzes their inventory levels. / Only products with managed stock are considered. / Each product’s stock quantity is compared against category-wise thresholds to determine whether it falls under: - Low stock - Very low (urgent) stock / The products are then separated into two arrays for further processing. |
| Format Low Stock Message | Code | Build Slack Block message for low stock | Separate products to low stock and very low stock | Send Low Stock Alert to Slack | This section prepares a formatted Slack message listing all products with low stock levels. / The alert includes: - Product name - Current stock - Category / The message is sent to the low stock Slack channel to help the team monitor inventory before it becomes critical. |
| Send Low Stock Alert to Slack | Slack | Post low stock alert to Slack channel | Format Low Stock Message | — | This section prepares a formatted Slack message listing all products with low stock levels. / The alert includes: - Product name - Current stock - Category / The message is sent to the low stock Slack channel to help the team monitor inventory before it becomes critical. |
| Format Urgent Low Stock Message for Jira | Code | Build Jira description text for urgent items | Separate products to low stock and very low stock | Create an Issue in Jira | This part handles products with very low (urgent) stock. / First, a Jira issue is created with: - A clear summary - High priority - Detailed product list in the description / After the Jira issue is created, a Slack alert is sent to the urgent stock channel, including: - Urgent low stock product details - Jira ticket ID - Jira ticket key / This ensures critical inventory issues are both tracked and immediately visible. |
| Create an Issue in Jira | Jira | Create high-priority issue for urgent items | Format Urgent Low Stock Message for Jira | Format Urgent Low Stock Message | This part handles products with very low (urgent) stock. / First, a Jira issue is created with: - A clear summary - High priority - Detailed product list in the description / After the Jira issue is created, a Slack alert is sent to the urgent stock channel, including: - Urgent low stock product details - Jira ticket ID - Jira ticket key / This ensures critical inventory issues are both tracked and immediately visible. |
| Format Urgent Low Stock Message | Code | Build Slack Block message for urgent items + Jira refs | Create an Issue in Jira | Send Urgent Low Stock Alert to Slack | This part handles products with very low (urgent) stock. / First, a Jira issue is created with: - A clear summary - High priority - Detailed product list in the description / After the Jira issue is created, a Slack alert is sent to the urgent stock channel, including: - Urgent low stock product details - Jira ticket ID - Jira ticket key / This ensures critical inventory issues are both tracked and immediately visible. |
| Send Urgent Low Stock Alert to Slack | Slack | Post urgent low stock alert to Slack channel | Format Urgent Low Stock Message | — | This part handles products with very low (urgent) stock. / First, a Jira issue is created with: - A clear summary - High priority - Detailed product list in the description / After the Jira issue is created, a Slack alert is sent to the urgent stock channel, including: - Urgent low stock product details - Jira ticket ID - Jira ticket key / This ensures critical inventory issues are both tracked and immediately visible. |
| Sticky Note | Sticky Note | Documentation | — | — | ## How It Works / This workflow automatically monitors product inventory levels in WooCommerce on a scheduled basis and sends alerts when stock falls below defined thresholds. / -> Every Monday at midnight, the workflow: / 1. Fetches all products from WooCommerce. / 2. Evaluates each product’s stock quantity based on category-specific thresholds. / 3. Separates products into: - Low stock - Very low (urgent) stock / 4. Sends: - A low stock alert to a dedicated Slack channel. - A very low stock alert that: - Creates a Jira issue - Sends a Slack alert including the Jira ticket details. / This ensures teams are proactively notified and urgent stock shortages are tracked and resolved quickly. / ## Setup Steps / 1. Configure Schedule Trigger - Set the Schedule Trigger to run every Monday at midnight. / 2. Connect WooCommerce - Add WooCommerce credentials. - Use the Get many products operation to fetch all products. / 3. Define Stock Thresholds - In the “Separate products to low stock and very low stock” Code node: - Set category-wise thresholds for: - Low stock - Urgent (very low) stock / 4. Configure Slack Integration - Connect your Slack workspace. - Select: - One channel for low stock alerts - One channel for urgent low stock alerts / 5. Configure Jira Integration - Connect Jira Software Cloud credentials. - Select: - Project - Issue type - Assignee - Priority / 6. Activate Workflow - Once activated, alerts and Jira issues will be generated automatically every week. |
| Sticky Note1 | Sticky Note | Documentation | — | — | Runs the workflow automatically every Monday at midnight without manual intervention. |
| Sticky Note10 | Sticky Note | Documentation | — | — | This part fetches all products from WooCommerce and analyzes their inventory levels. / Only products with managed stock are considered. / Each product’s stock quantity is compared against category-wise thresholds to determine whether it falls under: - Low stock - Very low (urgent) stock / The products are then separated into two arrays for further processing. |
| Sticky Note11 | Sticky Note | Documentation | — | — | This section prepares a formatted Slack message listing all products with low stock levels. / The alert includes: - Product name - Current stock - Category / The message is sent to the low stock Slack channel to help the team monitor inventory before it becomes critical. |
| Sticky Note12 | Sticky Note | Documentation | — | — | This part handles products with very low (urgent) stock. / First, a Jira issue is created with: - A clear summary - High priority - Detailed product list in the description / After the Jira issue is created, a Slack alert is sent to the urgent stock channel, including: - Urgent low stock product details - Jira ticket ID - Jira ticket key / This ensures critical inventory issues are both tracked and immediately visible. |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name: **Low-Stock Alerts to Slack** (or your preferred name).

2. **Add trigger: Schedule**
   - Add node: **Schedule Trigger**
   - Configure a weekly rule:
     - Interval: **Weeks = 1**
     - Day of week: **Monday**
     - Time: **00:00** (midnight) according to your n8n timezone

3. **Add WooCommerce product fetch**
   - Add node: **WooCommerce**
   - Operation: **Get All** (Products)
   - Credentials: create/select WooCommerce API credentials:
     - Store URL
     - Consumer Key / Consumer Secret
   - Connect: **Schedule Trigger → WooCommerce**

4. **Add classification Code node**
   - Add node: **Code** (JavaScript)
   - Name: **Separate products to low stock and very low stock**
   - Paste logic that:
     - Iterates through `$input.all()`
     - Skips items with `!manage_stock` or `stock_quantity === null`
     - Uses category slug if exactly one category; else default
     - Applies two threshold maps (low vs urgent)
     - Returns one item containing:
       - `low_stock` array
       - `uregent_low_stock` array (keep spelling consistent unless you update all dependent nodes)
   - Connect: **WooCommerce → Separate products…**

5. **Low stock path: format Slack blocks**
   - Add node: **Code**
   - Name: **Format Low Stock Message**
   - Build a Slack Block Kit payload in the node output (top-level `text` and `blocks`).
   - Include a guard clause: if `low_stock` array is empty, `return []`.
   - Connect: **Separate products… → Format Low Stock Message**

6. **Low stock path: Slack send**
   - Add node: **Slack**
   - Name: **Send Low Stock Alert to Slack**
   - Resource/operation: send a message to a **channel**
   - Message type: **Block**
   - Set blocks input to expression using the formatter output (as in the workflow: `={{ $json }}`)
   - Pick channel: your **low stock** channel
   - Credentials: Slack OAuth (bot token) with permissions to post
   - Connect: **Format Low Stock Message → Send Low Stock Alert to Slack**

7. **Urgent path: build Jira description**
   - Add node: **Code**
   - Name: **Format Urgent Low Stock Message for Jira**
   - Read `uregent_low_stock` from input and compose a plain-text description.
   - (Recommended) Add a guard clause: if urgent list empty, `return []` to avoid creating empty Jira issues.
   - Connect: **Separate products… → Format Urgent Low Stock Message for Jira**

8. **Urgent path: create Jira issue**
   - Add node: **Jira Software Cloud**
   - Name: **Create an Issue in Jira**
   - Operation: **Create Issue**
   - Configure:
     - Project: select your target project
     - Issue Type: select (e.g., Bug/Task)
     - Summary: `Urgent Low Stock - {{ $now.format('yyyy-MM-dd') }}`
     - Priority: High
     - Assignee: desired user (ensure project allows assigning)
     - Description: expression from previous node: `{{ $json.description }}`
   - Credentials: Jira Software Cloud OAuth/API token setup in n8n
   - Connect: **Format Urgent… for Jira → Create an Issue in Jira**

9. **Urgent path: format Slack blocks including Jira keys**
   - Add node: **Code**
   - Name: **Format Urgent Low Stock Message**
   - Retrieve urgent list via either:
     - direct input if you pass it along, or
     - node reference to the classifier node (as implemented): `$('Separate products to low stock and very low stock').first().json.uregent_low_stock`
   - Use Jira output from current `$input` (from Jira node) to embed:
     - `$input.first().json.id`
     - `$input.first().json.key`
   - Guard clause: if urgent list empty, `return []`.
   - Connect: **Create an Issue in Jira → Format Urgent Low Stock Message**

10. **Urgent path: Slack send**
   - Add node: **Slack**
   - Name: **Send Urgent Low Stock Alert to Slack**
   - Message type: **Block**
   - Blocks value: `={{ $json }}`
   - Channel: your **urgent** channel
   - Enable retry on fail (optional but recommended)
   - Connect: **Format Urgent Low Stock Message → Send Urgent Low Stock Alert to Slack**

11. **(Optional) Add sticky notes / documentation**
   - Add Sticky Notes describing each section for maintainability.

12. **Activate workflow**
   - Verify credentials for WooCommerce, Slack, Jira.
   - Execute once manually to confirm:
     - No Slack message is sent when arrays are empty.
     - Jira issue is only created when urgent items exist (if you added the recommended guard clause).
   - Activate.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| The workflow separates stock alerts into “Low” and “Very Low (Urgent)” using category-specific thresholds, then sends Slack alerts; urgent alerts also create a Jira issue and include Jira ticket details in Slack. | Sticky note “How It Works” (in-workflow documentation) |
| The urgent stock output key is spelled `uregent_low_stock`. Changing it requires updating all downstream references (Jira formatter + urgent Slack formatter). | Code-node contract / integration constraint |
| Slack messages are built with Block Kit `rich_text`. Large inventories may exceed Slack message limits; consider truncation or batching. | Slack API behavior |

