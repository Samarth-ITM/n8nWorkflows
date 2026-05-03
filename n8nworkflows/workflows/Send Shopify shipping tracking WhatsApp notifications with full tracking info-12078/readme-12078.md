Send Shopify shipping tracking WhatsApp notifications with full tracking info

https://n8nworkflows.xyz/workflows/send-shopify-shipping-tracking-whatsapp-notifications-with-full-tracking-info-12078


# Send Shopify shipping tracking WhatsApp notifications with full tracking info

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Send Shopify shipping tracking WhatsApp notifications with full tracking info

**Purpose:**  
When a Shopify fulfillment is created (order shipped), the workflow:
1) updates an internal PostgreSQL `orders` record with shipping/tracking data,  
2) looks up the customer in PostgreSQL,  
3) validates and normalizes the customer phone number to international format,  
4) detects the customer’s language (Arabic vs English),  
5) prevents sending messages to test/development phone numbers,  
6) sends a WhatsApp text message via the Meta WhatsApp Business API including full tracking info (tracking number, company, URL).

**Target use cases:**
- Automated “order shipped” notifications with tracking details
- Bilingual customer communication (AR/EN) using stored preferences
- Ensuring WhatsApp-compatible phone formatting and avoiding test-number spam

### 1.1 Shopify Event Reception
Receives the fulfillment event from Shopify (`fulfillments/create`) as the workflow’s entry point.

### 1.2 Database Update (Orders)
Marks the order as shipped in PostgreSQL and stores tracking company + tracking number.

### 1.3 Customer Lookup + Phone Normalization
Finds a customer record (via phone from the fulfillment payload), then validates/normalizes the phone number (adds country code if missing).

### 1.4 Language Routing + Test-Number Protection
Routes to Arabic vs English message based on customer language, and blocks messages for specific test numbers.

### 1.5 WhatsApp Notification Delivery
Sends a WhatsApp message via HTTP POST to Meta Graph API v18.0.

---

## 2. Block-by-Block Analysis

### Block 1 — Shopify Fulfillment Trigger
**Overview:** Captures “fulfillment created” events from Shopify and starts the workflow with the fulfillment payload.  
**Nodes involved:** `fulfilment`

#### Node: fulfilment
- **Type / role:** `Shopify Trigger` — webhook trigger for Shopify events.
- **Configuration (interpreted):**
  - **Topic:** `fulfillments/create`
  - **Authentication:** Shopify Access Token (private/custom app token)
- **Inputs / outputs:**
  - **Input:** none (entry point)
  - **Output:** fulfillment payload to `Update Order Shipped1`
- **Key data used later (from payload):**
  - `order_id`, `tracking_number`, `tracking_company`, `tracking_url`
  - Phone/country candidates under:
    - `destination.phone`, `destination.country_code`
    - `shipping_address.phone`, `shipping_address.country_code`
    - `phone`
- **Failure / edge cases:**
  - Shopify webhook delivery failures (network, signature/auth, disabled webhook)
  - Missing tracking fields if fulfillment created without tracking info
  - Phone not present in fulfillment payload (common in some shops)
- **Version requirements:** Node type version `1` (as provided); ensure n8n has Shopify Trigger available and configured.

---

### Block 2 — Update Order as Shipped (PostgreSQL)
**Overview:** Updates the internal `orders` row matching the Shopify order ID and saves shipping metadata.  
**Nodes involved:** `Update Order Shipped1`

#### Node: Update Order Shipped1
- **Type / role:** `Postgres` — executes an update query.
- **Configuration (interpreted):**
  - **Operation:** Execute Query
  - **Query intent:**
    - `status = 'shipped'`
    - `shipping_company = tracking_company` (with single-quote escaping)
    - `tracking_number = tracking_number`
    - Filter: `WHERE order_id = <Shopify order_id>`
    - `RETURNING customer_id` (returned but not used downstream)
  - Uses expressions referencing `$('fulfilment').item.json.*`
- **Key expressions / variables:**
  - Escaping: `(...tracking_company...).replace(/'/g, "''")`
  - `{{ $('fulfilment').item.json.order_id }}`
- **Inputs / outputs:**
  - **Input:** from `fulfilment`
  - **Output:** to `Select rows from a table6`
- **Credentials:** PostgreSQL credential named **“Postgres account”**
- **Failure / edge cases:**
  - SQL error if `order_id` is missing/non-numeric
  - Update affects 0 rows if the order is not yet in DB
  - `tracking_company` could be null/empty; query still runs
  - Potential SQL-injection risk is partially mitigated for `tracking_company` only; best practice is parameterized queries for all variables.
- **Version requirements:** Node type version `2`.

---

### Block 3 — Customer Lookup by Phone (PostgreSQL)
**Overview:** Finds a customer record in `customers` by phone number taken from the fulfillment payload, then fetches additional profile fields.  
**Nodes involved:** `Select rows from a table6`, `Get Customer1`

#### Node: Select rows from a table6
- **Type / role:** `Postgres` — table select operation (UI-driven).
- **Configuration (interpreted):**
  - **Operation:** Select
  - **Table:** `public.customers`
  - **Limit:** 1
  - **Where:** `phone = {{ $('fulfilment').item.json.destination.phone }}`
- **Inputs / outputs:**
  - **Input:** from `Update Order Shipped1`
  - **Output:** to `Phone Validator`
- **Credentials:** “Postgres account”
- **Failure / edge cases:**
  - If `destination.phone` is missing or formatted differently than DB (spaces, leading +, country code), no match → downstream may fail.
  - Limit 1 may pick an arbitrary match if phone is not unique.
  - Does not use the `customer_id` returned from the update query, so mismatches are possible.
- **Version requirements:** Node type version `2.6`

#### Node: Get Customer1
- **Type / role:** `Postgres` — execute query to retrieve customer properties.
- **Configuration (interpreted):**
  - **Operation:** Execute Query
  - **Query:** `SELECT phone, language, first_name FROM customers WHERE website_id = <website_id>`
  - `<website_id>` is taken from `Select rows from a table6`.item.json.website_id
- **Inputs / outputs:**
  - **Input:** from `Phone Validator`
  - **Output:** to `Check Language`
- **Credentials:** “Postgres account”
- **Failure / edge cases:**
  - If `Select rows...` returned no rows, `website_id` is undefined → SQL error.
  - If multiple rows share website_id, query returns multiple items; downstream “Get Customer1”.item.json may not be deterministic.
- **Version requirements:** Node type version `2`

---

### Block 4 — Phone Validation & Normalization (Code)
**Overview:** Normalizes the phone number into an international `+<countrycode><number>` format and marks invalid/empty numbers.  
**Nodes involved:** `Phone Validator`

#### Node: Phone Validator
- **Type / role:** `Code` — per-item JavaScript transformation.
- **Configuration (interpreted):**
  - **Mode:** Run Once for Each Item
  - **Logic:**
    - Defines a map of country codes for ~20 countries (OM, JO, AE, SA, …).
    - Pulls fulfillment payload via `$('fulfilment').item.json` (not from the immediate input item).
    - Extracts phone from (priority):
      1) `destination.phone`
      2) `shipping_address.phone`
      3) `phone`
    - Extracts country code from:
      - `destination.country_code` or `shipping_address.country_code`, default `OM`
    - Cleans phone: trims, removes spaces, removes all non-digits except `+`
    - If too short/empty: returns `validation_status: 'empty_or_invalid'`
    - Checks if number starts with any known country code (ignoring `+`)
      - If not: prepends default country code for detected country (fallback `968`)
      - Ensures `+` prefix exists
    - Returns `{ phone, original_phone, country_code, validation_status }`
- **Inputs / outputs:**
  - **Input:** from `Select rows from a table6` (but it mainly reads from `fulfilment`)
  - **Output:** to `Get Customer1`
- **Key variables returned:**
  - `phone` (normalized, usually `+...`)
  - `validation_status` (`success` or `empty_or_invalid`)
- **Failure / edge cases:**
  - If the fulfillment payload lacks phone entirely, status becomes `empty_or_invalid`, but the workflow still proceeds and may attempt WhatsApp send.
  - Country codes list is limited; unsupported country_code falls back to Oman (+968).
  - Uses `$('fulfilment').item.json` directly; if there are multiple items or if the trigger changes, this can misalign.
- **Version requirements:** Code node type version `2`

---

### Block 5 — Language Detection & Routing
**Overview:** Checks the customer’s language preference and routes to Arabic or English notification pipelines.  
**Nodes involved:** `Check Language`, `If10`, `If11`

#### Node: Check Language
- **Type / role:** `IF` — conditional routing.
- **Configuration (interpreted):**
  - Compares: first 2 letters of `language` (default `'en'`) to `'ar'`
  - Expression: `{{ ($('Get Customer1').item.json.language || 'en').trim().substring(0, 2) }}`
- **Outputs / connections:**
  - **True (language == ar):** to `If10`
  - **False:** to `If11`
- **Failure / edge cases:**
  - If `Get Customer1` returns no item, expression may fail.
  - Language values like `ar-OM` are handled (substring(0,2) => `ar`).
- **Version requirements:** Node type version `1`

#### Node: If10 (Test-number filter for Arabic path)
- **Type / role:** `IF` — blocks sending to certain test numbers.
- **Configuration (interpreted):**
  - OR condition: phone equals one of:
    - `=+96897666604`  *(note the leading `=` looks unintended and will likely never match)*
    - `+1234567890`
- **Outputs / connections:**
  - **True:** goes to `WhatsApp Shipped AR`
  - **False:** no connection (workflow ends)
- **Important logic note:** As wired, **matching a “test” number results in sending**, not blocking. If the intent is “prevent notifications to test numbers”, this IF is inverted.
- **Failure / edge cases:**
  - Mis-typed constant `=+968...` prevents match.
  - If phone is invalid/empty, comparisons won’t match; the workflow will end (no send) because false path is unconnected.
- **Version requirements:** Node type version `2.3`

#### Node: If11 (Test-number filter for English path)
- Same structure and issues as `If10`.
- **Outputs / connections:**
  - **True:** goes to `WhatsApp Shipped EN1`
  - **False:** no connection

---

### Block 6 — WhatsApp Message Send (Meta Graph API)
**Overview:** Sends WhatsApp text messages containing shipping and tracking details in the selected language.  
**Nodes involved:** `WhatsApp Shipped AR`, `WhatsApp Shipped EN1`

#### Node: WhatsApp Shipped AR
- **Type / role:** `HTTP Request` — POST to WhatsApp Business API endpoint.
- **Configuration (interpreted):**
  - **Method:** POST
  - **URL:** `https://graph.facebook.com/v18.0/819420977915834/messages`
  - **Auth:** Bearer token via n8n credential (`httpBearerAuth`) named **“whatsapp”**
  - **Body:** JSON
    - `messaging_product: "whatsapp"`
    - `to`: phone with `+` removed: `{{ $('Phone Validator').item.json.phone.replace('+', '') }}`
    - `type: "text"`
    - `text.body`: Arabic message including:
      - first name, order_id, tracking_number, tracking_company, tracking_url
      - delivery estimate 7–12 business days
- **Inputs / outputs:**
  - **Input:** from `If10`
  - **Output:** none
- **Failure / edge cases:**
  - 401/403 if token invalid or phone number ID unauthorized
  - 400 if `to` is not a valid WhatsApp number, or if business restrictions apply
  - Rate limits / throttling from Meta
  - If `tracking_url` is missing, message contains blank URL
- **Version requirements:** HTTP Request node version `4`

#### Node: WhatsApp Shipped EN1
- **Type / role:** `HTTP Request` — POST to same endpoint.
- **Configuration:** Similar to Arabic node, English body.
- **Critical content issue:** The provided English JSON body appears **truncated / malformed** (ends with `We will continue to keep you informed until your order is delivered.\nT  }`), which may cause:
  - invalid JSON (request fails before sending)
  - broken message content
- **Inputs / outputs:**
  - **Input:** from `If11`
  - **Output:** none
- **Failure / edge cases:** same as Arabic, plus JSON formatting failure risk.
- **Version requirements:** HTTP Request node version `4`

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| fulfilment | Shopify Trigger | Receives Shopify `fulfillments/create` webhook | — | Update Order Shipped1 | ---  ## 🎯 Key Features  - ✅ **Real-time Fulfillment Sync:** Instant database update when order ships  - ✅ **Complete Tracking Info:** Includes tracking number, company, and URL  - ✅ **Bilingual Support:** Auto-detects customer language preference from database  - ✅ **Phone Validation:** Handles 23+ country codes with auto-formatting  - ✅ **Test Protection:** Prevents notifications to development numbers  - ✅ **Personalization:** Uses customer first name in greeting  - ✅ **Delivery Expectations:** Sets clear 7-12 day timeline  - ✅ **Professional Branding:** Luxury brand tone in both languages  - ✅ **Trackable Links:** Includes clickable tracking URL  ---  ## 📊 Database Operations  ### **Orders Table Update**  - **Status Change:** 'open/paid' → 'shipped'  - **New Fields Added:**   - shipping_company: Courier service name   - tracking_number: Package tracking ID  - **Lookup Key:** order_id (from Shopify)  - **Returns:** customer_id for subsequent queries  ### **Customers Table Query**  - **Lookup:** By website_id (Shopify customer ID)  - **Retrieved:** phone, language, first_name  - **Purpose:** Get communication preferences  ---  ## 📞 Technical Details  - **WhatsApp API Version:** v18.0  - **API Endpoint:** graph.facebook.com/v18.0/819420977915834/messages  - **Authentication:** Bearer YOUR_TOKEN_HERE (credential: \"whatsapp mira\")  - **Phone Format:** International format with country code (+CountryCode + Number)  - **API Phone Format:** Removes '+' before sending to WhatsApp API  - **Database:** PostgreSQL  - **Credential Name:** \"Postgres account\"  - **Default Country:** Oman (+968) if country_code not detected  - **Test Numbers Blocked:**   - +96897666604 (Oman)   - +962798087441 (Jordan)  ---  ## 🔄 Data Flow Summary  1. **Fulfillment created** → Shopify webhook fires  2. **Update order status** → Mark as 'shipped' with tracking details  3. **Lookup customer** → Find by phone from fulfillment data  4. **Validate phone** → Format with country code  5. **Get preferences** → Retrieve language and name from database  6. **Detect language** → Route to AR or EN path  7. **Filter tests** → Skip internal phone numbers  8. **Send notification** → WhatsApp message with tracking info  ---  ## 💼 Business Logic  - **Delivery Window:** Consistently communicates 7-12 business days  - **Brand Voice:** Professional luxury brand tone  - **Customer Service:** Proactive communication at shipping milestone  - **Tracking Empowerment:** Provides all tools for self-service tracking  - **Multi-channel Sync:** Database, Shopify, and WhatsApp aligned  ---  ## 🌐 Multi-language Support  \n| Element | Arabic | English |\n|---------|--------|---------|\n| Greeting | مرحبًا | Hello |\n| Status | تم شحن طلبكم بنجاح | Successfully shipped |\n| Order No | رقم الطلب | Order No |\n| Tracking No | رقم التتبع | Tracking No |\n| Courier | شركة الشحن | Courier Company |\n| Delivery | من 7 إلى 12 يوم عمل | 7–12 business days | |
| Update Order Shipped1 | Postgres | Updates `orders` with shipped status and tracking fields | fulfilment | Select rows from a table6 | (same as above) |
| Select rows from a table6 | Postgres | Looks up a customer record by phone | Update Order Shipped1 | Phone Validator | (same as above) |
| Phone Validator | Code | Normalizes phone into international format and flags invalid | Select rows from a table6 | Get Customer1 | (same as above) |
| Get Customer1 | Postgres | Retrieves `phone, language, first_name` by `website_id` | Phone Validator | Check Language | (same as above) |
| Check Language | IF | Routes to Arabic vs English based on `language` | Get Customer1 | If10; If11 | (same as above) |
| If10 | IF | Test-number filter (Arabic path) | Check Language | WhatsApp Shipped AR | (same as above) |
| WhatsApp Shipped AR | HTTP Request | Sends Arabic WhatsApp shipped message | If10 | — | ### **8A. Arabic Shipping Notification - \"WhatsApp Shipped AR\"**  - **Type:** HTTP Request (POST)  - **Endpoint:** Facebook Graph API v18.0 (WhatsApp Business)  - **Phone Format:** Removes '+' prefix before API call  - **Message Content (Arabic):**  - **Greeting:** \"مرحبًا [first_name]\" (Hello)  - **Status:** \"تم شحن طلبكم بنجاح\" (Your order has been shipped successfully)  - **Order Details:**  - رقم الطلب (Order No): order_id  - رقم التتبع (Tracking No): tracking_number  - شركة الشحن (Courier): tracking_company  - رابط التتبع (Tracking URL): tracking_url  - **Delivery Time:** 7-12 business days (in Arabic)  - **Instructions:** Can track via tracking number  - **Tone:** Formal, professional Arabic suitable for luxury brand  - **Purpose:** Send comprehensive shipping notification to Arabic-speaking customers |
| If11 | IF | Test-number filter (English path) | Check Language | WhatsApp Shipped EN1 | (same as above) |
| WhatsApp Shipped EN1 | HTTP Request | Sends English WhatsApp shipped message | If11 | — | ### **8B. English Shipping Notification - \"WhatsApp Shipped EN1\"**  - **Type:** HTTP Request (POST)  - **Endpoint:** Same WhatsApp Business API  - **Phone Format:** Removes '+' prefix  - **Message Content (English):**  - **Greeting:** \"Hello [first_name]\"  - **Status:** \"Your order has been shipped successfully\"  - **Order Details:**  - Order No: order_id  - Tracking No: tracking_number  - Courier Company: tracking_company  - Tracking URL: tracking_url  - **Delivery Time:** 7-12 business days  - **Instructions:** Track shipment using tracking number  \"  - **Tone:** Professional, informative English  - **Purpose:** Send comprehensive shipping notification to English-speaking customers |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow** in n8n and name it:  
   `Send Shopify shipping tracking WhatsApp notifications with full tracking info`

2) **Add Trigger node: Shopify Trigger**
   - Node: **Shopify Trigger**
   - **Event/Topic:** `fulfillments/create`
   - **Authentication:** Access Token
   - Create/attach Shopify credentials (e.g., **“oman account”**) with:
     - Shop domain
     - Admin API access token (or app token with webhook permissions)
   - Save; n8n will generate a webhook ID internally.

3) **Add Postgres node: “Update Order Shipped1”**
   - Node: **Postgres**
   - **Operation:** Execute Query
   - **Credentials:** create/attach **“Postgres account”** (host, db, user, password, SSL as needed)
   - Use a query equivalent to:
     - Update `orders` table: set `status='shipped'`, set `shipping_company` from fulfillment `tracking_company`, set `tracking_number`, filter by `order_id`, return `customer_id`.
   - Connect: `fulfilment` → `Update Order Shipped1`

4) **Add Postgres node: “Select rows from a table6”**
   - Node: **Postgres**
   - **Operation:** Select
   - **Schema:** `public`
   - **Table:** `customers`
   - **Limit:** 1
   - **Where clause:** `phone` equals expression from fulfillment:  
     `{{ $('fulfilment').item.json.destination.phone }}`
   - Connect: `Update Order Shipped1` → `Select rows from a table6`

5) **Add Code node: “Phone Validator”**
   - Node: **Code**
   - **Mode:** Run once for each item
   - Paste logic implementing:
     - country code map
     - phone extraction from fulfillment payload
     - cleaning/normalizing and return fields: `phone`, `original_phone`, `country_code`, `validation_status`
   - Connect: `Select rows from a table6` → `Phone Validator`

6) **Add Postgres node: “Get Customer1”**
   - Node: **Postgres**
   - **Operation:** Execute Query
   - Query equivalent to:
     - `SELECT phone, language, first_name FROM customers WHERE website_id = {{ $('Select rows from a table6').item.json.website_id }}`
   - Connect: `Phone Validator` → `Get Customer1`

7) **Add IF node: “Check Language”**
   - Node: **IF**
   - Condition: string equals
     - Value1: `{{ ($('Get Customer1').item.json.language || 'en').trim().substring(0, 2) }}`
     - Value2: `ar`
   - Connect: `Get Customer1` → `Check Language`

8) **Add IF node: “If10” (Arabic path test filter)**
   - Node: **IF**
   - Add OR conditions comparing:
     - `{{ $('Phone Validator').item.json.phone }}` equals `+96897666604`
     - `{{ $('Phone Validator').item.json.phone }}` equals `+1234567890`
   - Connect: `Check Language` **true** output → `If10`

9) **Add IF node: “If11” (English path test filter)**
   - Same as step 8 (or adjust numbers as needed).
   - Connect: `Check Language` **false** output → `If11`

10) **Add HTTP Request node: “WhatsApp Shipped AR”**
   - Node: **HTTP Request**
   - **Method:** POST
   - **URL:** `https://graph.facebook.com/v18.0/819420977915834/messages`
   - **Authentication:** Bearer Auth credential
     - Create credential (e.g., “whatsapp”) with your permanent/long-lived token
   - **Send Body:** JSON
   - Body fields:
     - `messaging_product: "whatsapp"`
     - `to: {{ $('Phone Validator').item.json.phone.replace('+', '') }}`
     - `type: "text"`
     - `text.body`: Arabic message with variables (first_name, order_id, tracking_number, tracking_company, tracking_url)
   - Connect: `If10` **true** → `WhatsApp Shipped AR`

11) **Add HTTP Request node: “WhatsApp Shipped EN1”**
   - Same endpoint/auth/body structure, English message text.
   - Ensure the JSON is valid and message text is complete.
   - Connect: `If11` **true** → `WhatsApp Shipped EN1`

12) **(Recommended fixes while reproducing)**
   - **Invert test filter logic** to *block* test numbers:
     - Either connect the **false** output to WhatsApp nodes (send only if NOT test),
     - or change conditions to “does not equal” and combine accordingly.
   - **Handle invalid phones:** add an IF after `Phone Validator`:
     - if `validation_status != success` → stop or log
   - **Customer lookup robustness:** use `customer_id` from the update query, or normalize phone before querying customers to improve match rate.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| WhatsApp API endpoint used: `https://graph.facebook.com/v18.0/819420977915834/messages` | Meta WhatsApp Business Cloud API (Graph API v18.0) |
| Phone formatting: workflow removes the leading `+` before sending to WhatsApp (`to` must be digits only). | WhatsApp Cloud API expects E.164 digits without `+` in many implementations |
| Default country code fallback is Oman `+968` if `country_code` not detected or unsupported. | Implemented in `Phone Validator` |
| The workflow claims blocking test numbers (+96897666604, +962798087441), but implemented IF checks include `+1234567890` and a likely typo `=+96897666604`, and the routing currently sends on match (not block). | Consistency issue between sticky note and node configuration |
| English WhatsApp node body appears truncated/malformed and should be corrected to valid JSON before production use. | `WhatsApp Shipped EN1` node |

