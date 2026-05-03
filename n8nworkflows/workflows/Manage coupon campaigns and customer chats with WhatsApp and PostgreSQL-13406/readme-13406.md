Manage coupon campaigns and customer chats with WhatsApp and PostgreSQL

https://n8nworkflows.xyz/workflows/manage-coupon-campaigns-and-customer-chats-with-whatsapp-and-postgresql-13406


# Manage coupon campaigns and customer chats with WhatsApp and PostgreSQL

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Manage coupon campaigns and customer chats with WhatsApp and PostgreSQL

This workflow delivers two tightly coupled capabilities backed by PostgreSQL:

1) **A web-based admin dashboard** (served as HTML from n8n) to manage partner companies, coupons, settings, and to view a real-time inbox + conversation history.

2) **A WhatsApp Business bot** that:
- routes **regular customers** through an interactive “company → coupons → coupon details” flow,
- routes the **admin** (phone number stored in DB settings) to an interactive admin menu allowing CRUD operations via WhatsApp,
- persists state via a `sessions` table to support multi-step admin operations.

### 1.1 Dashboard UI Delivery
- Serves a Vue.js + Tailwind single-page interface from `/coupon-bot/dashboard`.
- The SPA calls internal REST-like endpoints implemented by other n8n Webhook nodes.

### 1.2 Database Lifecycle Tools
- Full schema creation/reset endpoint: `/coupon-bot/execute-init`.
- Migration/repair endpoint: `/coupon-bot/migrate`.

### 1.3 REST API Layer for Dashboard
Implements POST webhooks that behave like REST endpoints (method selected using `body.httpMethod`):
- `/coupon-bot/api/stats`
- `/coupon-bot/api/companies`
- `/coupon-bot/api/coupons`
- `/coupon-bot/api/settings`
- `/coupon-bot/api/inbox`
- `/coupon-bot/api/history`
- `/coupon-bot/api/customers`
- `/coupon-bot/api/records`

### 1.4 WhatsApp Bot Ingestion + Routing
- Receives Meta WhatsApp webhook events at `/whatsapp`.
- Extracts sender and message details, logs incoming messages, upserts customers, loads session state, then branches into:
  - **Admin flow** (interactive menu + session-driven actions),
  - **Customer flow** (browse companies/coupons and log records).

---

## 2. Block-by-Block Analysis

### Block 2.1 — Dashboard UI (HTML SPA)
**Overview:** Serves the full admin dashboard HTML (Vue 3 + Tailwind) from an n8n webhook. The SPA uses `fetch()` to call internal webhooks for data operations.

**Nodes Involved:**
- `Dashboard Webhook`
- `Respond with Dashboard HTML`

#### Node: Dashboard Webhook
- **Type/Role:** `Webhook` — HTTP entry point for the dashboard UI.
- **Config:** Path `coupon-bot/dashboard`, response mode **Response Node**.
- **Input/Output:** Entry node → `Respond with Dashboard HTML`.
- **Failure modes:** n8n webhook not publicly reachable; reverse proxy path mismatch; missing base URL.
- **Version:** typeVersion 1.

#### Node: Respond with Dashboard HTML
- **Type/Role:** `RespondToWebhook` — returns the HTML SPA.
- **Config choices:**
  - `Content-Type: text/html`
  - Body contains Vue app with endpoints like:
    - `POST /webhook/coupon-bot/api/stats`
    - `POST /webhook/coupon-bot/api/companies` (with `{ httpMethod: 'GET' }` etc.)
    - `POST /webhook/coupon-bot/execute-init`, `POST /webhook/coupon-bot/migrate`
- **Key variables:** none in n8n, but frontend expects stable webhook paths.
- **Failure modes:** Large HTML payload; mixed-content issues if behind HTTPS; SPA assumes `/webhook/` prefix exists.
- **Version:** typeVersion 1.

---

### Block 2.2 — Full Database Initialization (Danger Zone)
**Overview:** Drops and recreates all core tables (factory reset) via a POST webhook.

**Nodes Involved:**
- `Execute SQL Webhook`
- `Postgres Init`
- `Respond Success`

#### Node: Execute SQL Webhook
- **Type/Role:** `Webhook` — triggers full DB reset.
- **Config:** `POST coupon-bot/execute-init`, response via response node.
- **Connections:** → `Postgres Init`.
- **Failure modes:** Anyone with access can wipe DB (no auth here); should be protected (IP allowlist, auth, or disabled in production).

#### Node: Postgres Init
- **Type/Role:** `Postgres` — executes schema SQL.
- **Config:** `executeQuery` with multi-statement SQL:
  - Drops: `coupons, companies, messages, records, customers, settings, sessions`
  - Creates tables: `customers, companies, coupons, records, messages, settings, sessions`
- **Credentials:** `Postgres account 2` (must point to the same DB used by the dashboard & bot).
- **Failure modes:** Permission errors (DROP/CREATE); transaction/statement execution limits; existing connections.
- **Version:** typeVersion 1.

#### Node: Respond Success
- **Type/Role:** `RespondToWebhook` — returns success JSON.
- **Output:** `{"status":"success","message":"Database initialized fully with correct schema."}`
- **Failure modes:** If Postgres node fails, response node won’t run unless error handling is added.

---

### Block 2.3 — Schema Migration / Repair
**Overview:** Repairs/extends existing schema without deleting data; adds missing columns/constraints and ensures `sessions` exists.

**Nodes Involved:**
- `Migrate API Webhook`
- `Repair Tables Schema`
- `Respond Migration Success`

#### Node: Migrate API Webhook
- **Type/Role:** `Webhook` — triggers migration.
- **Config:** `POST coupon-bot/migrate`, response via response node.
- **Security risk:** Unauthenticated schema manipulation.

#### Node: Repair Tables Schema
- **Type/Role:** `Postgres` — executes conditional ALTER/CREATE statements.
- **Config highlights:**
  - Adds `customers.phone` if missing, enforces unique constraint `customers_phone_unique`
  - Adds missing columns to `messages` (`sender_name`, `message_type`, `direction`)
  - Ensures timestamps exist, normalizes types
  - `CREATE TABLE IF NOT EXISTS sessions (...)`
  - Cleans corrupted settings rows: `DELETE FROM settings WHERE value LIKE '{{%';`
- **Credentials:** `Postgres account 2`
- **Failure modes:** Permissions; lock contention; incompatible existing types.

#### Node: Respond Migration Success
- **Type/Role:** Respond JSON `{"status":"success","message":"Schema updated successfully"}`

---

### Block 2.4 — Dashboard Stats API
**Overview:** Returns counts for customers/messages/records/companies used on dashboard home screen.

**Nodes Involved:**
- `Stats API Webhook`
- `Get Stats Query`
- `Respond Stats`

#### Node: Stats API Webhook
- **Type/Role:** `Webhook`
- **Config:** `POST coupon-bot/api/stats`, response via response node.

#### Node: Get Stats Query
- **Type/Role:** `Postgres` — SELECT with subqueries.
- **Output fields:** `customers`, `messages`, `records`, `companies`.
- **Credentials:** `Postgres account 2`

#### Node: Respond Stats
- **Type/Role:** `RespondToWebhook`
- **Config:** returns `JSON.stringify($items().map(i => i.json))`
- **Edge case:** Always returns an array (even though single row). Frontend expects array and uses `data[0]`.

---

### Block 2.5 — Companies CRUD API
**Overview:** Implements GET list and POST/PUT/DELETE modifications controlled by `body.httpMethod`.

**Nodes Involved:**
- `Companies API`
- `Switch Company Method`
- `Get Companies`
- `Modify Company`
- `Respond Companies`
- `Respond Company Modified`

#### Node: Companies API
- **Type/Role:** `Webhook`
- **Config:** `POST coupon-bot/api/companies`, response node.

#### Node: Switch Company Method
- **Type/Role:** `If`
- **Logic:** If `body.httpMethod != 'GET'` → modification path; else → listing path.
- **Expression:** `={{ $json.body.httpMethod }}`
- **Failure modes:** If body missing `httpMethod`, condition evaluates to `notEqual` and routes to modify, likely breaking.

#### Node: Get Companies
- **Type/Role:** `Postgres` list query.
- **SQL:** `SELECT * FROM companies ORDER BY company_id DESC;`

#### Node: Modify Company
- **Type/Role:** `Postgres` dynamic SQL builder in expression.
- **Behavior:**
  - POST: INSERT name/info
  - PUT: UPDATE name/info by `company_id`
  - else: DELETE by `company_id`
- **Major edge case:** SQL injection risk due to string concatenation without escaping (only `info` defaults). Names are not escaped.
- **Credentials:** `Postgres account 2`

#### Node: Respond Companies
- **Type/Role:** `RespondToWebhook`
- **Response:** If result has `company_id` returns JSON array else `"[]"`.

#### Node: Respond Company Modified
- **Type/Role:** `RespondToWebhook`
- **Response:** `JSON.stringify($json)`.

---

### Block 2.6 — Coupons CRUD API
**Overview:** Similar to Companies API; GET lists coupons with company name; POST inserts coupon; DELETE removes coupon.

**Nodes Involved:**
- `Coupons API`
- `Switch Coupon Method`
- `Get Coupons`
- `Modify Coupon`
- `Respond Coupons`
- `Respond Coupon Modified`

#### Node: Coupons API
- **Webhook:** `POST coupon-bot/api/coupons`

#### Node: Switch Coupon Method
- **If:** `body.httpMethod != 'GET'` → modify; else → get list.

#### Node: Get Coupons
- **Postgres:** `SELECT c.*, co.name as company_name FROM coupons c JOIN companies co ...`

#### Node: Modify Coupon
- **Postgres:** expression-built SQL:
  - POST: INSERT with many optional flags and expiry handling
  - else: DELETE by `coupon_id`
- **Edge cases:**
  - SQL injection risk: `coupon_text`, `other_info` concatenated without escaping quotes consistently.
  - Boolean insertion relies on JS booleans → Postgres accepts `true/false` text, generally ok.
  - Expiry date string inserted as `'YYYY-MM-DD'` if provided; type is TIMESTAMP; implicit cast depends on locale/format.

#### Node: Respond Coupons / Respond Coupon Modified
- Same pattern as companies.

---

### Block 2.7 — Inbox & Chat History APIs (Dashboard)
**Overview:** Powers the dashboard inbox list (latest message per conversation) and chat history (up to 100 messages).

**Nodes Involved:**
- `Inbox API` → `Get Messages` → `Respond Messages`
- `Chat History API` → `Get Chat History` → `Respond History`

#### Node: Inbox API
- **Webhook:** `POST coupon-bot/api/inbox`

#### Node: Get Messages
- **Postgres:** Uses `SELECT DISTINCT ON (COALESCE(c.phone, m.sender_phone))` to get latest message per phone.
- **Edge cases:** If customer linkage missing, uses sender_phone; sorting relies on created_at.

#### Node: Respond Messages
- **Response:** JSON array of rows.

#### Node: Chat History API
- **Webhook:** `POST coupon-bot/api/history`
- **Body requirement:** `customer_phone`

#### Node: Get Chat History
- **Postgres expression:** Normalizes `+` by `REPLACE(..., '+','')` and filters on provided phone (also stripped).
- **Limit:** 100 messages ordered ascending.
- **Edge case:** If phone missing/empty → query matches empty string and returns none.

#### Node: Respond History
- Returns JSON array.

---

### Block 2.8 — Settings API (Dashboard)
**Overview:** Stores and retrieves system settings: `admin_phone`, `welcome_message`, `end_message`.

**Nodes Involved:**
- `Settings API`
- `Switch Settings Method`
- `Save Settings`
- `Get Settings`
- `Respond Settings Saved`
- `Respond Settings`

#### Node: Settings API
- **Webhook:** `POST coupon-bot/api/settings`

#### Node: Switch Settings Method
- **If:** `body.httpMethod != 'GET'` → save, else → get.

#### Node: Save Settings
- **Postgres:** Three UPSERT statements concatenated:
  - `admin_phone` stored without escaping
  - welcome/end messages escaped with `.replace(/'/g,"''")`
- **alwaysOutputData:** true (ensures downstream responds even if query returns no rows).
- **Failure modes:** If messages contain unusual encodings; large values.

#### Node: Get Settings
- **Postgres:** `SELECT * FROM settings;`
- **alwaysOutputData:** true

#### Respond nodes
- Saved: `JSON.stringify($json)`
- Get: returns array or `"[]"`.

---

### Block 2.9 — Customers & Records APIs (Dashboard)
**Overview:** Exposes customer list and coupon redemption records.

**Nodes Involved:**
- `Customers API` → `Get Customers` → `Respond Customers`
- `Records API` → `Get Records` → `Respond Records`

#### Node: Customers API
- **Webhook:** `POST coupon-bot/api/customers`

#### Node: Get Customers
- **Postgres:** `SELECT * FROM customers ORDER BY created_at DESC;`

#### Node: Records API
- **Webhook:** `POST coupon-bot/api/records`

#### Node: Get Records
- **Postgres join:** records + customers + companies + coupons; ordered newest first.

#### Respond nodes
- Return JSON arrays.

---

### Block 2.10 — WhatsApp Webhook Intake, Normalization, Logging
**Overview:** Receives WhatsApp events, confirms they are message events, loads admin phone setting, extracts message content (text/button/list), upserts customer, logs message, loads session state, then routes admin vs customer.

**Nodes Involved:**
- `WhatsApp Webhook`
- `Is Message?`
- `Get Admin Phone`
- `Set Basic Data`
- `Upsert Customer`
- `Save Incoming Message`
- `Get User Session`
- `Is Admin?`

#### Node: WhatsApp Webhook
- **Type/Role:** `Webhook`
- **Config:** `POST whatsapp`, response mode response node.
- **Important:** Meta verification handshake is not implemented here (no GET verify flow); this only handles POST events.

#### Node: Is Message?
- **Type/Role:** `If`
- **Condition:** checks if payload contains `...value.messages`.
- **Expression:** `={{ $('WhatsApp Webhook').item.json.body.entry[0].changes[0].value.messages ? true : false }}`
- **Edge cases:** Any schema variation breaks expression (undefined indexes).

#### Node: Get Admin Phone
- **Postgres:** `SELECT * FROM settings WHERE key = 'admin_phone';`
- **alwaysOutputData:** true (so Set Basic Data can still run with missing row).

#### Node: Set Basic Data
- **Type:** `Set`
- **Extracts:**
  - `sender_phone`, `sender_name`, `phone_number_id`
  - `message_text` from multiple possible WhatsApp message structures
  - `message_type`
  - `button_id` from interactive replies
  - `admin_phone` from settings query (`$json.value`)
  - `is_admin` boolean compares sender to `admin_phone`
- **Edge cases:** If settings empty, `admin_phone` undefined and `is_admin` always false.

#### Node: Upsert Customer
- **Postgres:** Inserts/updates customer by phone; escapes name quotes; returns `customer_id`.
- **Edge case:** Phone uniqueness required; if schema broken, insert fails.

#### Node: Save Incoming Message
- **Postgres:** Inserts into `messages` as `INCOMING`.
- **Escaping:** message_text and sender_name escaped for `'`.
- **alwaysOutputData:** true

#### Node: Get User Session
- **Postgres:** `SELECT * FROM sessions WHERE phone = '{sender_phone}';`
- **alwaysOutputData:** true (session may not exist).

#### Node: Is Admin?
- **If:** routes:
  - **true** → `Check Admin State`
  - **false** → `Switch User Action` (customer path)

---

### Block 2.11 — Customer WhatsApp Browsing Flow (Companies → Coupons → Details)
**Overview:** Regular users can browse companies then coupons via interactive lists; coupon selection sends details and logs redemption record.

**Nodes Involved:**
- `Switch User Action`
- `Get Welcome & End Messages`
- `Get Companies List`
- `Send Companies List`
- `Log Sent Companies`
- `Is Company Selection?`
- `Get Coupons List`
- `Check If Single Coupon`
- `Send Coupons List`
- `Log Sent Coupons`
- `Get Coupon Details`
- `Log Coupon Redemption`
- `Get End Message For Coupon`
- `Send Coupon Details`
- `Log Sent Details`

#### Node: Switch User Action
- **If (any):** message type is `button_reply` or `interactive`.
  - True → `Is Company Selection?` (i.e., user clicked something)
  - False → `Get Companies List` (default entry: show companies)
- **Edge case:** Some WhatsApp events use `interactive` with nested types; handled.

#### Node: Get Welcome & End Messages
- **Postgres:** reads `welcome_message` and `end_message` for customizing replies.
- **Connection:** feeds into `Get Companies List` so message sender can include welcome message.

#### Node: Get Companies List
- **Postgres:** selects up to 10 companies (excluding name `'Add Company'`).
- **alwaysOutputData:** true to allow “no companies” reply.

#### Node: Send Companies List
- **HTTP Request (WhatsApp Graph API):**
  - POST `https://graph.facebook.com/v22.0/{phone_number_id}/messages`
  - Bearer auth credential `whatsapp`
  - Sends interactive list (max 10 rows) or fallback text if none.
  - Includes welcome message from settings.
- **executeOnce:** true (important: only executes once per workflow execution in some contexts; can cause unexpected behavior if multiple items).

#### Node: Log Sent Companies
- **Postgres:** logs an OUTGOING text summary (“Welcome! Choose…” or “Sorry, no companies…”).

#### Node: Is Company Selection?
- **If:** checks whether `button_id` starts with `company_`.
  - True → `Get Coupons List`
  - False → `Get Coupon Details` (assumes it’s `coupon_{id}`)
- **Edge case:** If `button_id` empty, `.startsWith` can fail; node uses `($('Set Basic Data').item.json.button_id.startsWith(...))` without guarding, so empty string is needed (it sets default `''`, good).

#### Node: Get Coupons List
- **Postgres:** selects coupons for company, filters expired: `(expiry_date IS NULL OR expiry_date > NOW())`, top 10 by value desc.

#### Node: Check If Single Coupon
- **If:** if `$items().length == 1`
  - True → `Get Coupon Details` (auto-open details)
  - False → `Send Coupons List`
- **Note:** This assumes Postgres returns one item when one coupon exists; if it returns 0 items, length 0 → goes to Send Coupons List, which handles empty.

#### Node: Send Coupons List
- **HTTP Request:** sends interactive list of coupons (max 10) or fallback text.

#### Node: Log Sent Coupons
- **Postgres:** logs OUTGOING summary (“Choose a coupon:” or “Sorry, no coupons…”).

#### Node: Get Coupon Details
- **Postgres:** loads selected coupon by `coupon_id`.
- **Connections:** → `Log Coupon Redemption`, `Get End Message For Coupon`, `Log Sent Details`.

#### Node: Log Coupon Redemption
- **Postgres:** inserts into `records` using customer phone lookup.
- **Edge cases:** If customer does not exist, insert prevented by `WHERE EXISTS`.

#### Node: Get End Message For Coupon
- **Postgres:** `SELECT value FROM settings WHERE key = 'end_message';`

#### Node: Send Coupon Details
- **HTTP Request:** sends text containing coupon code/value/notes and the end message.
- **Uses:** `$('Get Coupon Details').first().json...` and `$json.value` (end_message).
- **Edge cases:** Missing end_message row → defaults to hardcoded message.

#### Node: Log Sent Details
- **Postgres:** logs OUTGOING “Coupon Details...” (static “Thank you for choosing us!” in log, not exactly the sent end_message).

---

### Block 2.12 — Admin WhatsApp Flow (Menu + Session-Based Operations)
**Overview:** Admin is identified by matching sender phone to `settings.admin_phone`. Admin can manage companies/coupons and edit welcome/end messages via interactive lists and multi-step session states persisted in `sessions`.

**Nodes Involved (routing and menu):**
- `Check Admin State`
- `Admin State Router`
- `Admin Choice`
- `Show Admin Menu`
- `Log Admin Menu`

**Nodes Involved (admin states & actions):**
- Company add:
  - `Set Admin State (Add Co)` → `Ask Admin Input` → `Log Admin Input`
  - `Add Company (Admin)` → `Send Admin Success` → `Log Admin Success`
- Company edit:
  - `Set Edit Company List State` → `Get Companies For Edit` → `Send Edit Companies List`
  - selection → `Set Edit Company State` → `Ask Edit Company Name`
  - `Save Edited Company` → `Send Edit Company Success`
- Company delete:
  - `Set Delete Company List State` → `Get Companies For Delete` → `Send Delete Companies List`
  - selection → `Delete Company Exec` → `Send Delete Company Success`
- Coupon add:
  - `Set Add Coupon State` → `Get Companies For Add Coupon` → `Send Add Coupon Companies`
  - selection → `Set Coupon Company` → `Ask Coupon Text`
  - `Save Coupon Text` → `Ask Coupon Value`
  - `Insert New Coupon` → `Send Add Coupon Success`
- Coupon edit:
  - `Set Edit Coupon State` → `Get Coupons For Edit` → `Send Edit Coupons List`
  - selection → `Set Edit Coupon Select` → `Ask Edit Coupon Text`
  - `Save Edited Coupon` → `Send Edit Coupon Success`
- Coupon delete:
  - `Set Delete Coupon State` → `Get Coupons For Delete` → `Send Delete Coupons List`
  - selection → `Delete Coupon Exec` → `Send Delete Coupon Success`
- Edit messages:
  - `Set Edit Welcome State` → `Ask Welcome Message` → `Save Welcome Message` → `Send Welcome Success`
  - `Set Edit End State` → `Ask End Message` → `Save End Message` → `Send End Success`
- Admin “Browse as member”:
  - `admin_browse` route sends companies list (via `Get Companies List`)

#### Node: Check Admin State
- **If:** checks if session state is one of many “WAITING_*” states and current interaction is not an `admin_` command.
- **Goal:** If admin is mid-flow, route to `Admin State Router`; otherwise handle new admin command via `Admin Choice`.
- **Edge cases:** Session row may be missing (state undefined) → condition false, falls to `Admin Choice`.

#### Node: Admin State Router
- **Switch:** routes based on `$('Get User Session').item.json.state` (or `$json.state` for EDIT_WELCOME/EDIT_END due to inconsistent reference).
- **Important:** This switch expects session `state` and sometimes expects `data.company_id` / `data.coupon_id`.
- **Edge cases:** If `data` is JSONB but returned as string in n8n, `.company_id` may be undefined. (Depends on pg driver settings.)

#### Node: Admin Choice
- **Switch:** routes by `button_id`:
  - `admin_add_company`, `admin_browse`, `admin_edit_company`, `admin_delete_company`,
  - `admin_add_coupon`, `admin_edit_coupon`, `admin_delete_coupon`,
  - `admin_edit_welcome`, `admin_edit_end`,
  - and also a “startsWith company_/coupon_” rule.
  - fallback goes to `Show Admin Menu`.
- **Edge cases:** Ordering matters; a generic “true” rule exists before welcome/end rules in the rules list (there is a condition `{ leftValue: true == true }`). In many switch implementations, the first match wins—this can make later rules unreachable depending on n8n switch behavior. Verify in your n8n version: if rules are evaluated top-to-bottom, “always true” should be last.

#### Admin SQL nodes using credential mismatch
Several admin action Postgres nodes use credential named **`discounts DB`** instead of **`Postgres account 2`**:
- `Add Company (Admin)`
- `Save Edited Company`
- `Delete Company Exec`
- `Set Coupon Company`
- `Save Coupon Text`
- `Insert New Coupon`
- `Set Edit Coupon Select`
- `Save Edited Coupon`
- `Delete Coupon Exec`
- `Set Edit Welcome State`
- `Save Welcome Message`
- `Set Edit End State`
- `Save End Message`
This is a critical reproducibility detail: **both credentials must point to the same physical database** or you will create inconsistent state (dashboard sees one DB, admin modifies another).

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Dashboard Webhook | Webhook | Serves dashboard entrypoint | — | Respond with Dashboard HTML | # 🎯 Coupon Bot Dashboard\nA complete admin dashboard...\nSetup steps... ⚠️ Run /migrate first if upgrading existing database |
| Respond with Dashboard HTML | RespondToWebhook | Returns Vue/Tailwind HTML | Dashboard Webhook | — | 📊 Dashboard UI\n- Vue.js 3 + Tailwind CSS interface...\nServed via RespondToWebhook node |
| Execute SQL Webhook | Webhook | Triggers full DB reset/init | — | Postgres Init | ⚙️ System Tools\n- Database initialization (/execute-init)\n- Schema migration & repair (/migrate)\n- Full reset capability\n- Settings management |
| Postgres Init | Postgres | Drops & recreates schema | Execute SQL Webhook | Respond Success | 🗄️ Database Schema\n- Customers, Companies, Coupons tables...\nMigration support for updates |
| Respond Success | RespondToWebhook | Returns init success JSON | Postgres Init | — | ⚙️ System Tools\n- Database initialization (/execute-init)... |
| Stats API Webhook | Webhook | Stats endpoint for dashboard | — | Get Stats Query | 🔌 REST API Routes\n- GET/POST /api/companies...\n- GET /api/stats, /api/inbox, /api/history |
| Get Stats Query | Postgres | Counts customers/messages/records/companies | Stats API Webhook | Respond Stats | 🔌 REST API Routes\n- GET /api/stats... |
| Respond Stats | RespondToWebhook | Returns stats JSON array | Get Stats Query | — | 🔌 REST API Routes\n- GET /api/stats... |
| Companies API | Webhook | Companies CRUD endpoint | — | Switch Company Method | 🔌 REST API Routes\n- GET/POST /api/companies |
| Switch Company Method | If | Routes GET vs modify | Companies API | Modify Company; Get Companies | 🔌 REST API Routes\n- GET/POST /api/companies |
| Get Companies | Postgres | Lists companies | Switch Company Method | Respond Companies | 🔌 REST API Routes\n- GET/POST /api/companies |
| Modify Company | Postgres | Inserts/updates/deletes company | Switch Company Method | Respond Company Modified | 🔌 REST API Routes\n- GET/POST /api/companies |
| Respond Companies | RespondToWebhook | Returns companies list | Get Companies | — | 🔌 REST API Routes\n- GET/POST /api/companies |
| Respond Company Modified | RespondToWebhook | Returns modified company row | Modify Company | — | 🔌 REST API Routes\n- GET/POST /api/companies |
| Coupons API | Webhook | Coupons CRUD endpoint | — | Switch Coupon Method | 🔌 REST API Routes\n- GET/POST /api/coupons |
| Switch Coupon Method | If | Routes GET vs modify | Coupons API | Modify Coupon; Get Coupons | 🔌 REST API Routes\n- GET/POST /api/coupons |
| Get Coupons | Postgres | Lists coupons with company name | Switch Coupon Method | Respond Coupons | 🔌 REST API Routes\n- GET/POST /api/coupons |
| Modify Coupon | Postgres | Inserts/deletes coupon | Switch Coupon Method | Respond Coupon Modified | 🔌 REST API Routes\n- GET/POST /api/coupons |
| Respond Coupons | RespondToWebhook | Returns coupons list | Get Coupons | — | 🔌 REST API Routes\n- GET/POST /api/coupons |
| Respond Coupon Modified | RespondToWebhook | Returns modified coupon row | Modify Coupon | — | 🔌 REST API Routes\n- GET/POST /api/coupons |
| Inbox API | Webhook | Inbox list endpoint | — | Get Messages | 💬 Customer Inbox System\n- Real-time chat interface...\nWebhook endpoints: /api/inbox, /api/history |
| Get Messages | Postgres | Latest message per conversation | Inbox API | Respond Messages | 💬 Customer Inbox System\n- Real-time chat interface... |
| Respond Messages | RespondToWebhook | Returns inbox JSON array | Get Messages | — | 💬 Customer Inbox System\n- Real-time chat interface... |
| Settings API | Webhook | Settings get/save endpoint | — | Switch Settings Method | ⚙️ System Tools\n- Settings management |
| Switch Settings Method | If | Routes GET vs save | Settings API | Save Settings; Get Settings | ⚙️ System Tools\n- Settings management |
| Save Settings | Postgres | Upserts admin/welcome/end messages | Switch Settings Method | Respond Settings Saved | ⚙️ System Tools\n- Settings management |
| Get Settings | Postgres | Reads all settings | Switch Settings Method | Respond Settings | ⚙️ System Tools\n- Settings management |
| Respond Settings Saved | RespondToWebhook | Returns save result | Save Settings | — | ⚙️ System Tools\n- Settings management |
| Respond Settings | RespondToWebhook | Returns settings list | Get Settings | — | ⚙️ System Tools\n- Settings management |
| Migrate API Webhook | Webhook | Triggers schema repair | — | Repair Tables Schema | ⚙️ System Tools\n- Schema migration & repair (/migrate) |
| Repair Tables Schema | Postgres | Alters tables/adds constraints | Migrate API Webhook | Respond Migration Success | 🗄️ Database Schema\n- Migration support for updates |
| Respond Migration Success | RespondToWebhook | Returns migration success JSON | Repair Tables Schema | — | ⚙️ System Tools\n- Schema migration & repair (/migrate) |
| Chat History API | Webhook | Conversation history endpoint | — | Get Chat History | 💬 Customer Inbox System\n- Webhook endpoints: /api/inbox, /api/history |
| Get Chat History | Postgres | Loads last 100 messages for phone | Chat History API | Respond History | 💬 Customer Inbox System\n- Message history tracking |
| Respond History | RespondToWebhook | Returns history JSON | Get Chat History | — | 💬 Customer Inbox System\n- Message history tracking |
| Customers API | Webhook | Customers list endpoint | — | Get Customers | 👥 Customer Management\n- API endpoints: /api/customers, /api/records |
| Get Customers | Postgres | Lists customers | Customers API | Respond Customers | 👥 Customer Management\n- Customer profiles and phone tracking |
| Respond Customers | RespondToWebhook | Returns customers JSON | Get Customers | — | 👥 Customer Management\n- API endpoints: /api/customers |
| Records API | Webhook | Redemption records endpoint | — | Get Records | 👥 Customer Management\n- Coupon usage records |
| Get Records | Postgres | Lists redemption records with joins | Records API | Respond Records | 👥 Customer Management\n- Usage analytics and history |
| Respond Records | RespondToWebhook | Returns records JSON | Get Records | — | 👥 Customer Management\n- API endpoints: /api/records |
| WhatsApp Webhook | Webhook | Receives WhatsApp events | — | Is Message? | # 🤖 WhatsApp Coupon Bot\nComplete WhatsApp Business API integration... |
| Is Message? | If | Validates message presence | WhatsApp Webhook | Get Admin Phone | # 🔌 WhatsApp API Integration & User Routing\nMessage Types... |
| Get Admin Phone | Postgres | Loads admin_phone setting | Is Message? | Set Basic Data | # 🔌 WhatsApp API Integration & User Routing\nUser Detection... |
| Set Basic Data | Set | Normalizes sender/message fields | Get Admin Phone | Upsert Customer | # 🔌 WhatsApp API Integration & User Routing\nCustomer Path / Admin Path... |
| Upsert Customer | Postgres | Insert/update customer by phone | Set Basic Data | Save Incoming Message | 👥 Customer Management\n- Customer profiles and phone tracking |
| Save Incoming Message | Postgres | Logs INCOMING message | Upsert Customer | Get User Session | 💬 Customer Inbox System\n- Message history tracking |
| Get User Session | Postgres | Loads conversation state | Save Incoming Message | Is Admin? | # 🔄 Session-Based State Management\nBest practice: reset to IDLE... |
| Is Admin? | If | Routes admin vs customer flow | Get User Session | Check Admin State; Switch User Action | # 🔌 WhatsApp API Integration & User Routing\nRoutes to Admin Path or Customer Path |
| Switch User Action | If | Customer: interactive vs default browse | Is Admin? (false) | Is Company Selection?; Get Companies List | # 🔌 WhatsApp API Integration & User Routing\nCustomer Path: Show welcome → List companies... |
| Get Welcome & End Messages | Postgres | Reads welcome/end texts | — | Get Companies List | # 🤖 WhatsApp Coupon Bot\nAdmin Setup: Set admin_phone in dashboard... |
| Get Companies List | Postgres | Lists companies for user | Switch User Action; Get Welcome & End Messages; Admin Choice (browse) | Send Companies List; Log Sent Companies | # 🔌 WhatsApp API Integration & User Routing\nList not showing: verify max 10 items... |
| Send Companies List | HTTP Request | Sends interactive list to WhatsApp | Get Companies List | — | # 🔌 WhatsApp API Integration & User Routing\nAPI Endpoint: POST https://graph.facebook.com/v22.0/{phone_number_id}/messages |
| Log Sent Companies | Postgres | Logs OUTGOING summary | Get Companies List | — | 💬 Customer Inbox System\n- Message history tracking |
| Is Company Selection? | If | Detects company_ vs coupon_ selection | Switch User Action; Admin Choice | Get Coupons List; Get Coupon Details | # 🔌 WhatsApp API Integration & User Routing\nInteractive List - 10 items max |
| Get Coupons List | Postgres | Lists coupons for chosen company | Is Company Selection? | Check If Single Coupon; Log Sent Coupons | # 🔌 WhatsApp API Integration & User Routing\nList not showing: verify max 10 items |
| Check If Single Coupon | If | Auto-open coupon if only one | Get Coupons List | Get Coupon Details; Send Coupons List | # 🔌 WhatsApp API Integration & User Routing\nCustomer Path... |
| Send Coupons List | HTTP Request | Sends coupon list to WhatsApp | Check If Single Coupon | — | # 🔌 WhatsApp API Integration & User Routing\nInteractive messages (lists, buttons) |
| Log Sent Coupons | Postgres | Logs OUTGOING summary | Get Coupons List | — | 💬 Customer Inbox System\n- Message history tracking |
| Get Coupon Details | Postgres | Loads coupon details by id | Is Company Selection?; Check If Single Coupon | Log Coupon Redemption; Get End Message For Coupon; Log Sent Details | 👥 Customer Management\n- Coupon usage records |
| Log Coupon Redemption | Postgres | Inserts redemption record | Get Coupon Details | — | 👥 Customer Management\n- Coupon usage records |
| Get End Message For Coupon | Postgres | Loads end_message | Get Coupon Details | Send Coupon Details | # 🤖 WhatsApp Coupon Bot\nSettings: end message |
| Send Coupon Details | HTTP Request | Sends coupon details text | Get End Message For Coupon | — | # 🔌 WhatsApp API Integration & User Routing\nText - 4096 chars max |
| Log Sent Details | Postgres | Logs OUTGOING coupon details | Get Coupon Details | — | 💬 Customer Inbox System\n- Message history tracking |
| Check Admin State | If | Routes ongoing admin sessions | Is Admin? (true) | Admin State Router; Admin Choice | # 🔄 Session-Based State Management\nSessions persist until operation completes |
| Admin State Router | Switch | Routes by session state | Check Admin State | (various admin SQL nodes) | # 🔄 Session-Based State Management\nCommon states... |
| Admin Choice | Switch | Routes admin menu actions | Check Admin State | (state setters, lists, show menu, edits) | # 🔌 WhatsApp API Integration & User Routing\nAdmin Path: Show menu → Manage... |
| Show Admin Menu | HTTP Request | Sends admin list menu | Admin Choice | — | # 🤖 WhatsApp Coupon Bot\nAdmin gets full management menu |
| Log Admin Menu | Postgres | Logs OUTGOING menu | Admin Choice | — | 💬 Customer Inbox System\n- Message history tracking |
| Set Admin State (Add Co) | Postgres | Sets session WAITING_COMPANY_NAME | Admin Choice | Ask Admin Input; Log Admin Input | # 🔄 Session-Based State Management\nAlways reset to IDLE... |
| Ask Admin Input | HTTP Request | Prompts for company name | Set Admin State (Add Co) | — | # 🔌 WhatsApp API Integration & User Routing\nAdmin management flow |
| Log Admin Input | Postgres | Logs prompt message | Set Admin State (Add Co) | — | 💬 Customer Inbox System\n- Message history tracking |
| Add Company (Admin) | Postgres | Inserts company and resets session | Admin State Router | Send Admin Success; Log Admin Success | # 🔄 Session-Based State Management\nReset to IDLE after operation |
| Send Admin Success | HTTP Request | Confirms company added | Add Company (Admin) | — | # 🔌 WhatsApp API Integration & User Routing |
| Log Admin Success | Postgres | Logs confirmation | Add Company (Admin) | — | 💬 Customer Inbox System |
| Set Edit Company List State | Postgres | Sets WAITING_EDIT_COMPANY_SELECT | Admin Choice | Get Companies For Edit | # 🔄 Session-Based State Management |
| Get Companies For Edit | Postgres | Lists companies for editing | Set Edit Company List State | Send Edit Companies List | # 🔌 WhatsApp API Integration & User Routing |
| Send Edit Companies List | HTTP Request | Sends list with editco_ ids | Get Companies For Edit | — | # 🔌 WhatsApp API Integration & User Routing |
| Set Edit Company State | Postgres | Sets WAITING_EDIT_COMPANY_NAME + company_id | Admin State Router | Ask Edit Company Name | # 🔄 Session-Based State Management |
| Ask Edit Company Name | HTTP Request | Prompts new name | Set Edit Company State | — | # 🔌 WhatsApp API Integration & User Routing |
| Save Edited Company | Postgres | Updates company + resets session | Admin State Router | Send Edit Company Success | # 🔄 Session-Based State Management |
| Send Edit Company Success | HTTP Request | Confirms edit | Save Edited Company | — | # 🔌 WhatsApp API Integration & User Routing |
| Set Delete Company List State | Postgres | Sets WAITING_DELETE_COMPANY_SELECT | Admin Choice | Get Companies For Delete | # 🔄 Session-Based State Management |
| Get Companies For Delete | Postgres | Lists companies for deletion | Set Delete Company List State | Send Delete Companies List | # 🔌 WhatsApp API Integration & User Routing |
| Send Delete Companies List | HTTP Request | Sends list with delco_ ids | Get Companies For Delete | — | # 🔌 WhatsApp API Integration & User Routing |
| Delete Company Exec | Postgres | Deletes company + resets session | Admin State Router | Send Delete Company Success | # 🔄 Session-Based State Management |
| Send Delete Company Success | HTTP Request | Confirms deletion | Delete Company Exec | — | # 🔌 WhatsApp API Integration & User Routing |
| Set Add Coupon State | Postgres | Sets WAITING_ADD_COUPON_COMPANY | Admin Choice | Get Companies For Add Coupon | # 🔄 Session-Based State Management |
| Get Companies For Add Coupon | Postgres | Lists companies to attach coupon | Set Add Coupon State | Send Add Coupon Companies | # 🔌 WhatsApp API Integration & User Routing |
| Send Add Coupon Companies | HTTP Request | Sends list with addcpn_ ids | Get Companies For Add Coupon | — | # 🔌 WhatsApp API Integration & User Routing |
| Set Coupon Company | Postgres | Sets WAITING_ADD_COUPON_TEXT + data.company_id | Admin State Router | Ask Coupon Text | # 🔄 Session-Based State Management |
| Ask Coupon Text | HTTP Request | Prompts coupon code | Set Coupon Company | — | # 🔌 WhatsApp API Integration & User Routing |
| Save Coupon Text | Postgres | Sets WAITING_ADD_COUPON_VALUE + data.coupon_text | Admin State Router | Ask Coupon Value | # 🔄 Session-Based State Management |
| Ask Coupon Value | HTTP Request | Prompts numeric value or skip | Save Coupon Text | — | # 🔌 WhatsApp API Integration & User Routing |
| Insert New Coupon | Postgres | Inserts coupon, resets session | Admin State Router | Send Add Coupon Success | # 🔄 Session-Based State Management |
| Send Add Coupon Success | HTTP Request | Confirms coupon added | Insert New Coupon | — | # 🔌 WhatsApp API Integration & User Routing |
| Set Edit Coupon State | Postgres | Sets WAITING_EDIT_COUPON_SELECT | Admin Choice | Get Coupons For Edit | # 🔄 Session-Based State Management |
| Get Coupons For Edit | Postgres | Lists coupons with company name | Set Edit Coupon State | Send Edit Coupons List | # 🔌 WhatsApp API Integration & User Routing |
| Send Edit Coupons List | HTTP Request | Sends list with editcpn_ ids | Get Coupons For Edit | — | # 🔌 WhatsApp API Integration & User Routing |
| Set Edit Coupon Select | Postgres | Sets WAITING_EDIT_COUPON_TEXT + coupon_id | Admin State Router | Ask Edit Coupon Text | # 🔄 Session-Based State Management |
| Ask Edit Coupon Text | HTTP Request | Prompts new coupon code | Set Edit Coupon Select | — | # 🔌 WhatsApp API Integration & User Routing |
| Save Edited Coupon | Postgres | Updates coupon + resets session | Admin State Router | Send Edit Coupon Success | # 🔄 Session-Based State Management |
| Send Edit Coupon Success | HTTP Request | Confirms edit | Save Edited Coupon | — | # 🔌 WhatsApp API Integration & User Routing |
| Set Delete Coupon State | Postgres | Sets WAITING_DELETE_COUPON_SELECT | Admin Choice | Get Coupons For Delete | # 🔄 Session-Based State Management |
| Get Coupons For Delete | Postgres | Lists coupons to delete | Set Delete Coupon State | Send Delete Coupons List | # 🔌 WhatsApp API Integration & User Routing |
| Send Delete Coupons List | HTTP Request | Sends list with delcpn_ ids | Get Coupons For Delete | — | # 🔌 WhatsApp API Integration & User Routing |
| Delete Coupon Exec | Postgres | Deletes coupon + resets session | Admin State Router | Send Delete Coupon Success | # 🔄 Session-Based State Management |
| Send Delete Coupon Success | HTTP Request | Confirms delete | Delete Coupon Exec | — | # 🔌 WhatsApp API Integration & User Routing |
| Set Edit Welcome State | Postgres | Sets state EDIT_WELCOME | Admin Choice | Ask Welcome Message | # 🔄 Session-Based State Management |
| Ask Welcome Message | HTTP Request | Prompts new welcome text | Set Edit Welcome State | — | # 🔌 WhatsApp API Integration & User Routing |
| Save Welcome Message | Postgres | Upserts welcome_message + resets session | Admin State Router | Send Welcome Success | # 🔄 Session-Based State Management |
| Send Welcome Success | HTTP Request | Confirms update | Save Welcome Message | — | # 🔌 WhatsApp API Integration & User Routing |
| Set Edit End State | Postgres | Sets state EDIT_END | Admin Choice | Ask End Message | # 🔄 Session-Based State Management |
| Ask End Message | HTTP Request | Prompts new end text | Set Edit End State | — | # 🔌 WhatsApp API Integration & User Routing |
| Save End Message | Postgres | Upserts end_message + resets session | Admin State Router | Send End Success | # 🔄 Session-Based State Management |
| Send End Success | HTTP Request | Confirms update | Save End Message | — | # 🔌 WhatsApp API Integration & User Routing |
| Sticky Note | StickyNote | Comment | — | — | # 🎯 Coupon Bot Dashboard\nA complete admin dashboard...\n⚠️ Run /migrate first if upgrading existing database |
| Sticky Note1 | StickyNote | Comment | — | — | 📊 Dashboard UI\n- Vue.js 3 + Tailwind CSS interface... |
| Sticky Note2 | StickyNote | Comment | — | — | 🗄️ Database Schema\n- Customers, Companies, Coupons tables... |
| Sticky Note3 | StickyNote | Comment | — | — | 🔌 REST API Routes\n- GET/POST /api/companies... |
| Sticky Note4 | StickyNote | Comment | — | — | ⚙️ System Tools\n- Database initialization (/execute-init)... |
| Sticky Note5 | StickyNote | Comment | — | — | 💬 Customer Inbox System\n- Real-time chat interface... |
| Sticky Note6 | StickyNote | Comment | — | — | 👥 Customer Management\n- Customer profiles and phone tracking... |
| Sticky Note8 | StickyNote | Comment | — | — | # 🤖 WhatsApp Coupon Bot\nComplete WhatsApp Business API integration... |
| Sticky Note9 | StickyNote | Comment | — | — | # 🔄 Session-Based State Management\nThe bot uses PostgreSQL sessions table... |
| Sticky Note10 | StickyNote | Comment | — | — | # 🔌 WhatsApp API Integration & User Routing\nMessage Types... Common issues... |
| Sticky Note7 | StickyNote | Comment | — | — | ![Dashboard Screenshot](https://jobotai.site/1.png) |
| Sticky Note11 | StickyNote | Comment | — | — | ![Dashboard Screenshot](https://jobotai.site/4.png) |
| Sticky Note12 | StickyNote | Comment | — | — | ![Dashboard Screenshot](https://jobotai.site/5.png) |
| Sticky Note13 | StickyNote | Comment | — | — | # 🎯 Coupon Bot  +  Dashboard + Inbox |

> Note: Sticky notes are listed as nodes above; they don’t connect to other nodes.

---

## 4. Reproducing the Workflow from Scratch

1) **Create PostgreSQL credentials**
   - Create two credentials (to match the workflow names):
     - `Postgres account 2`
     - `discounts DB`
   - Point both to the **same database** (host, db, user, password).
   - Ensure the DB user can `CREATE`, `ALTER`, `DROP` tables.

2) **Create WhatsApp Bearer credential**
   - Create credential `httpBearerAuth` named `whatsapp`
   - Set Bearer token = WhatsApp Cloud API access token.

3) **Build the Dashboard UI endpoint**
   1. Add `Webhook` node named **Dashboard Webhook**
      - Path: `coupon-bot/dashboard`
      - Method: GET (default)
      - Response Mode: **Response Node**
   2. Add `Respond to Webhook` named **Respond with Dashboard HTML**
      - Respond With: **Text**
      - Add header `Content-Type = text/html`
      - Paste the provided HTML (Vue/Tailwind app).
   3. Connect: `Dashboard Webhook` → `Respond with Dashboard HTML`

4) **Build DB initialization endpoint**
   1. Add `Webhook` **Execute SQL Webhook**
      - Path: `coupon-bot/execute-init`
      - Method: POST
      - Response Mode: Response Node
   2. Add `Postgres` **Postgres Init**
      - Operation: Execute Query
      - Credentials: `Postgres account 2`
      - Paste schema SQL (DROP + CREATE statements).
   3. Add `Respond to Webhook` **Respond Success**
      - Respond with JSON success payload.
   4. Connect: Webhook → Postgres Init → Respond Success

5) **Build migration endpoint**
   1. Add `Webhook` **Migrate API Webhook** (POST `coupon-bot/migrate`)
   2. Add `Postgres` **Repair Tables Schema** with migration SQL
   3. Add `Respond to Webhook` **Respond Migration Success**
   4. Connect: Webhook → Postgres → Respond

6) **Create REST endpoints for dashboard**
   - For each below: `Webhook (POST, response node)` → `Postgres` → `RespondToWebhook`
   1. **Stats**
      - Path `coupon-bot/api/stats`
      - Query counts
      - Respond: JSON array
   2. **Companies**
      - Path `coupon-bot/api/companies`
      - Add `If` node **Switch Company Method** on `body.httpMethod != 'GET'`
      - GET path: `Get Companies` query
      - Modify path: `Modify Company` query (POST/PUT/DELETE based on `body.httpMethod`)
      - Two respond nodes: list vs modified
   3. **Coupons**
      - Path `coupon-bot/api/coupons`
      - Same pattern as companies (GET list with join; POST insert; DELETE)
   4. **Settings**
      - Path `coupon-bot/api/settings`
      - If `httpMethod != GET`: `Save Settings` (UPSERT admin/welcome/end)
      - Else: `Get Settings`
   5. **Inbox**
      - Path `coupon-bot/api/inbox` → query latest message per phone
   6. **History**
      - Path `coupon-bot/api/history` → query by `body.customer_phone`
   7. **Customers**
      - Path `coupon-bot/api/customers` → list customers
   8. **Records**
      - Path `coupon-bot/api/records` → list joined redemption records

7) **Create WhatsApp ingestion webhook**
   1. Add `Webhook` **WhatsApp Webhook**
      - Path: `whatsapp`
      - Method: POST
      - Response Mode: Response Node (you may optionally respond 200 OK with empty body; current workflow doesn’t explicitly respond—consider adding).
   2. Add `If` **Is Message?**
      - Condition checks presence of `entry[0].changes[0].value.messages`
   3. Add `Postgres` **Get Admin Phone** (settings lookup)
   4. Add `Set` **Set Basic Data** to extract:
      - sender phone, sender name, message text, message type, button id, phone_number_id, admin_phone, is_admin
   5. Add `Postgres` **Upsert Customer**
   6. Add `Postgres` **Save Incoming Message**
   7. Add `Postgres` **Get User Session**
   8. Add `If` **Is Admin?** and connect:
      - true → Admin block
      - false → Customer block

8) **Customer WhatsApp flow**
   1. Add `If` **Switch User Action** for interactive vs default
   2. Add `Postgres` **Get Welcome & End Messages**
   3. Add `Postgres` **Get Companies List**
   4. Add `HTTP Request` **Send Companies List**
      - POST `https://graph.facebook.com/v22.0/{{phone_number_id}}/messages`
      - Auth: Bearer `whatsapp`
      - Body: interactive list JSON
   5. Add `Postgres` **Log Sent Companies**
   6. Add `If` **Is Company Selection?**
   7. Add `Postgres` **Get Coupons List**
   8. Add `If` **Check If Single Coupon**
   9. Add `HTTP Request` **Send Coupons List**
   10. Add `Postgres` **Log Sent Coupons**
   11. Add `Postgres` **Get Coupon Details**
   12. Add `Postgres` **Log Coupon Redemption**
   13. Add `Postgres` **Get End Message For Coupon**
   14. Add `HTTP Request` **Send Coupon Details**
   15. Add `Postgres` **Log Sent Details**
   - Connect nodes in the same order as in the JSON connections.

9) **Admin WhatsApp flow (session-driven)**
   1. Add `If` **Check Admin State**
   2. Add `Switch` **Admin State Router** keyed on session state
   3. Add `Switch` **Admin Choice** keyed on `button_id`
   4. Add menu sender nodes (`Show Admin Menu`, `Log Admin Menu`)
   5. Add state setters (`Set Admin State (Add Co)`, etc.) using `sessions` UPSERT patterns
   6. Add list sender nodes for edit/delete selections (companies and coupons)
   7. Add final execution nodes (insert/update/delete + session reset to `IDLE`)
   - Ensure all admin Postgres nodes use the correct DB credential (ideally one unified credential).

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Coupon Bot Dashboard overview, setup steps, warning to run migrate first when upgrading | (Sticky Note content) |
| Dashboard screenshots | https://jobotai.site/1.png |
| Dashboard screenshots | https://jobotai.site/4.png |
| Dashboard screenshots | https://jobotai.site/5.png |
| WhatsApp Graph API endpoint used | `POST https://graph.facebook.com/v22.0/{phone_number_id}/messages` |
| WhatsApp list limitations referenced | Lists: max 10 items; Text: 4096 chars (not enforced in workflow) |
| Security note (implicit): init/migrate endpoints unauthenticated | Consider adding auth, IP restrictions, or disable in production |
| SQL safety note (implicit): multiple nodes build SQL via string concatenation | Use parameterized queries or sanitize inputs to prevent SQL injection |

If you want, I can also provide: (a) a corrected “Admin Choice” switch ordering to avoid the always-true rule shadowing later routes, and (b) a hardening plan (auth + parameterized SQL) without changing functional behavior.