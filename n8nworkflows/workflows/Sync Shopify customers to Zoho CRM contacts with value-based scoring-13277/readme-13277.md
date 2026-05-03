Sync Shopify customers to Zoho CRM contacts with value-based scoring

https://n8nworkflows.xyz/workflows/sync-shopify-customers-to-zoho-crm-contacts-with-value-based-scoring-13277


# Sync Shopify customers to Zoho CRM contacts with value-based scoring

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:** Synchronize Shopify customer activity (typically from `customers/create`, optionally `orders/create`) into **Zoho CRM Contacts**, and compute/update value-based engagement metrics (order total, order count, lifetime spend) to support segmentation and sales prioritization.

**Target use cases:**
- Keep Zoho CRM contacts aligned with Shopify customer/order events.
- Maintain basic “value scoring” (lifetime spend, order frequency).
- Avoid duplicates by searching Zoho first, then updating or creating.

### 1.1 Trigger & Global Configuration
Receives Shopify webhook events and sets global thresholds used later for segmentation (high-value order tag threshold and lifetime spend threshold).

### 1.2 Data Extraction & Enrichment
Normalizes incoming Shopify payload into a consistent set of computed fields (order totals, lifetime spend, boolean high-value flag, etc.).

### 1.3 Zoho Lookup & Conditional Sync
Attempts to find an existing Zoho Contact and routes execution:
- If found: update custom fields.
- If not found: create a new contact.

---

## 2. Block-by-Block Analysis

### Block 1 — Trigger & Global Configuration

**Overview:** Listens to Shopify events and defines workflow-wide numeric thresholds used to calculate high-value status and lifetime spend segmentation.

**Nodes involved:**
- Trigger on New Customer or Order
- Workflow Configuration

#### Node: Trigger on New Customer or Order
- **Type / role:** `Shopify Trigger` — webhook-based entry point from Shopify.
- **Configuration (interpreted):**
  - **Topic:** `customers/create` (configured)
  - **Authentication:** Access Token (Shopify Private App / Admin token style)
  - Uses a stored webhookId (managed by n8n).
- **Inputs / outputs:**
  - **Input:** none (trigger)
  - **Output:** Shopify event JSON (customer created payload; pinned data resembles an order payload, see edge cases).
- **Credentials:** `shopifyAccessTokenApi` (Shopify Access Token account 4)
- **Version notes:** Node `typeVersion: 1`
- **Edge cases / failures:**
  - **Webhook topic mismatch:** Sticky note suggests `orders/create` is also applicable, but the node is set to `customers/create`. If you expect order fields (like `total_price`, `fulfillments`), use `orders/create` or adjust extraction logic.
  - **Auth/permissions:** invalid token, missing webhook scopes, or store not reachable.
  - **Payload shape differences:** `customers/create` payload differs from `orders/create`; downstream nodes currently assume order-like fields.

#### Node: Workflow Configuration
- **Type / role:** `Set` — defines reusable configuration values.
- **Configuration (interpreted):**
  - Adds:
    - `minOrderValueForHighValueTag` = **500**
    - `lifetimeSpendThreshold` = **1000**
  - **Include other fields:** enabled (passes through incoming JSON plus these fields).
- **Key expressions / variables:** none (constants)
- **Inputs / outputs:**
  - **Input:** Trigger output
  - **Output:** Same data + configuration fields
- **Version notes:** `typeVersion: 3.4`
- **Edge cases / failures:**
  - Minimal risk; only potential issues if downstream nodes reference this node but execution path changes.

---

### Block 2 — Data Extraction & Enrichment

**Overview:** Computes normalized customer/order metrics from the Shopify payload to drive CRM updates and segmentation.

**Nodes involved:**
- Extract Customer Data

#### Node: Extract Customer Data
- **Type / role:** `Set` — computes derived fields (scoring inputs).
- **Configuration (interpreted):**
  - **Include other fields:** enabled
  - Adds computed fields:
    - `orderTotal` (number): `={{ $json.total_price || 0 }}`
    - `orderCount` (number): `={{ $json.customer?.orders_count || 1 }}`
    - `lifetimeSpend` (number): `={{ $json.customer?.total_spent || $json.total_price || 0 }}`
    - `customerTags` (string): `={{ $json.customer?.tags || '' }}`
    - `isHighValue` (boolean):  
      `={{ ($json.total_price || 0) >= $('Workflow Configuration').first().json.minOrderValueForHighValueTag }}`
    - `shopifyCustomerId` (string): `={{ $json.customer?.id || $json.id }}`
- **Inputs / outputs:**
  - **Input:** Workflow Configuration
  - **Output:** Enriched event data for Zoho lookup
- **Version notes:** `typeVersion: 3.4`
- **Edge cases / failures:**
  - **Numeric parsing:** Shopify `total_price` is often a string (e.g., `"100.00"`). JS comparison and arithmetic can behave unexpectedly. Consider `Number($json.total_price)` and `Number($json.customer?.total_spent)` to avoid string/number issues.
  - **Topic mismatch:** For `customers/create`, `total_price` may not exist; the workflow defaults to `0`, which could incorrectly mark customers as not high value and set spend to 0.
  - **Missing nested fields:** Uses optional chaining for `customer.*` which is safe, but assumes at least something exists for `shopifyCustomerId`.

---

### Block 3 — Zoho Lookup & Conditional Sync

**Overview:** Queries Zoho CRM for an existing contact and routes to update vs. create actions.

**Nodes involved:**
- Search for Existing Contact
- Contact Exists?
- Update Existing Contact
- Create New Contact

#### Node: Search for Existing Contact
- **Type / role:** `Zoho CRM` — retrieves contacts to determine if the customer already exists.
- **Configuration (interpreted):**
  - **Resource:** Contact
  - **Operation:** Get All
  - **Limit:** 2
  - **Fields requested:** `Email`
  - **Important:** No search criteria/filter is configured.
- **Inputs / outputs:**
  - **Input:** Extract Customer Data
  - **Output:** Up to 2 Zoho contact records (first page of contacts), not necessarily matching the Shopify customer.
- **Credentials:** `zohoOAuth2Api` (Zoho account 9)
- **Version notes:** `typeVersion: 1`
- **Edge cases / failures:**
  - **Logical correctness risk (major):** Because it uses **Get All** without filtering by email/Shopify ID, it will return arbitrary contacts. The workflow may update the wrong contact or think a contact “exists” when it’s unrelated.
  - **Auth errors / expired token** with Zoho OAuth2.
  - **Rate limits** if volume is high.
- **Recommended fix:** Use a Zoho search operation (or criteria query) by **Email** (preferred) or a custom field storing `shopifyCustomerId`.

#### Node: Contact Exists?
- **Type / role:** `IF` — decides whether an existing contact was found.
- **Configuration (interpreted):**
  - Condition: checks whether `$('Search for Existing Contact').item.json.id` **exists**.
- **Inputs / outputs:**
  - **Input:** Search for Existing Contact
  - **Output (true branch / index 0):** Update Existing Contact
  - **Output (false branch / index 1):** Create New Contact
- **Version notes:** `typeVersion: 2.2`
- **Edge cases / failures:**
  - If Search returns multiple items, the IF evaluates **per item**. That can lead to multiple updates/creates unexpectedly.
  - If Search returns an empty list, downstream “false” branch may not run as intended unless n8n produces an empty-item flow. A common mitigation is to use a node that always returns one item with “found/not found” status.

#### Node: Update Existing Contact
- **Type / role:** `Zoho CRM` — updates custom fields on an existing contact.
- **Configuration (interpreted):**
  - **Resource:** Contact
  - **Operation:** Update
  - **Contact ID:** `={{ $json.id }}` (expects the incoming item to be a Zoho contact record)
  - Updates custom fields:
    - `Engagement_Score` = `Extract Customer Data.lifetimeSpend`
    - `Mentions_Counts` = `Extract Customer Data.orderCount`
  - (Field IDs are provided as Zoho field API identifiers: `Engagement_Score`, `Mentions_Counts`)
- **Inputs / outputs:**
  - **Input:** Contact Exists? (true path)
  - **Output:** Updated contact record / API response
- **Credentials:** `zohoOAuth2Api` (Zoho account 9)
- **Version notes:** `typeVersion: 1`
- **Edge cases / failures:**
  - **Wrong contact updated** due to unfiltered search upstream.
  - **Field ID mismatch:** if custom fields do not exist or API names differ, Zoho will reject the update.
  - **Type mismatch:** Zoho may expect numeric values; ensure extracted values are numbers.
  - **Permissions:** OAuth user must have rights to edit Contacts and those fields.

#### Node: Create New Contact
- **Type / role:** `Zoho CRM` — creates a new contact when none exists.
- **Configuration (interpreted):**
  - **Resource:** Contact
  - **Operation:** Create (implied by the node name and parameters; operation field not explicitly shown in the snippet but node is configured as a create-style node)
  - **Last Name:**  
    `={{ $('Extract Customer Data').item.json.fulfillments[0].line_items[0].name }}`
  - **Additional fields:** empty
  - **Always output data:** enabled (helps keep flow outputs even if API returns minimal data)
- **Inputs / outputs:**
  - **Input:** Contact Exists? (false path)
  - **Output:** Created contact record / API response
- **Credentials:** `zohoOAuth2Api` (Zoho account 9)
- **Version notes:** `typeVersion: 1`
- **Edge cases / failures:**
  - **Invalid last name source:** `fulfillments[0].line_items[0].name` exists for order payloads but not for customer payloads. For `customers/create`, this likely fails (expression error) or returns undefined.
  - **Zoho required fields:** Zoho Contacts typically require `Last_Name`; if the expression resolves empty, creation fails.
  - **Duplicates:** No email/unique key is set, so duplicates are likely even with a proper search.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Trigger on New Customer or Order | Shopify Trigger | Entry point: receive Shopify webhook event | — | Workflow Configuration | ## Shopify Trigger & Set Configuration\nThe process begins with a Shopify Trigger listening for new customers or orders created in your store. This is followed by a Workflow Configuration node that establishes global operational parameters, such as the minimum order value required for high-value tagging and lifetime spend thresholds. |
| Workflow Configuration | Set | Define global thresholds used downstream | Trigger on New Customer or Order | Extract Customer Data | ## Shopify Trigger & Set Configuration\nThe process begins with a Shopify Trigger listening for new customers or orders created in your store. This is followed by a Workflow Configuration node that establishes global operational parameters, such as the minimum order value required for high-value tagging and lifetime spend thresholds. |
| Extract Customer Data | Set | Compute order/customer metrics (spend, count, flags) | Workflow Configuration | Search for Existing Contact | ## Data Extraction & CRM Search\nThis stage processes the raw shopify data by using the Extract Customer Data node to calculate key engagement metrics, such as lifetime spend, order frequency and high-value customer status. These details are then passed to the Search for Existing Contact node, which queries Zoho CRM to check if the customer already exists in your database, ensuring that activity is recorded against the correct profile without creating duplicates. |
| Search for Existing Contact | Zoho CRM | Lookup Zoho contacts prior to upsert | Extract Customer Data | Contact Exists? | ## Data Extraction & CRM Search\nThis stage processes the raw shopify data by using the Extract Customer Data node to calculate key engagement metrics, such as lifetime spend, order frequency and high-value customer status. These details are then passed to the Search for Existing Contact node, which queries Zoho CRM to check if the customer already exists in your database, ensuring that activity is recorded against the correct profile without creating duplicates. |
| Contact Exists? | IF | Route to update vs create | Search for Existing Contact | Update Existing Contact; Create New Contact | ## Conditional Routing & CRM Synchronization\nThe final stage uses a Contact Exists? IF node to determine the appropriate path in Zoho CRM based on whether a matching record was found. If the contact is new, the workflow routes to the Create New Contact node to generate a fresh profile using the Shopify data. If the contact already exists, it routes to the Update Existing Contact node, which automatically synchronizes current Shopify metrics—such as engagement scores and order counts—directly to the customer's CRM record. |
| Update Existing Contact | Zoho CRM | Update Zoho contact custom fields (scoring) | Contact Exists? (true) | — | ## Conditional Routing & CRM Synchronization\nThe final stage uses a Contact Exists? IF node to determine the appropriate path in Zoho CRM based on whether a matching record was found. If the contact is new, the workflow routes to the Create New Contact node to generate a fresh profile using the Shopify data. If the contact already exists, it routes to the Update Existing Contact node, which automatically synchronizes current Shopify metrics—such as engagement scores and order counts—directly to the customer's CRM record. |
| Create New Contact | Zoho CRM | Create new Zoho contact | Contact Exists? (false) | — | ## Conditional Routing & CRM Synchronization\nThe final stage uses a Contact Exists? IF node to determine the appropriate path in Zoho CRM based on whether a matching record was found. If the contact is new, the workflow routes to the Create New Contact node to generate a fresh profile using the Shopify data. If the contact already exists, it routes to the Update Existing Contact node, which automatically synchronizes current Shopify metrics—such as engagement scores and order counts—directly to the customer's CRM record. |

Sticky note present in the canvas but not tied to a single node row (general workflow note):
- **How It Works / Setup Steps** content is included in section 5.

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow**
   - Name it: **(Retail) Shopify to CRM Contact Sync** (or your preferred name).
   - Keep it inactive until credentials and webhook are tested.

2) **Add Shopify Trigger**
   - Add node: **Shopify Trigger**
   - Set **Authentication**: *Access Token*
   - Select/create Shopify credentials:
     - Create **Shopify Access Token** credential with:
       - Store URL / shop domain
       - Admin access token
   - Set **Topic**:
     - If you want order metrics from the payload (total price, fulfillments), prefer: `orders/create`
     - If you want only customer creation events, use: `customers/create` (but adjust later nodes accordingly)
   - Save the node so n8n generates/registers the webhook.

3) **Add “Workflow Configuration” (Set node)**
   - Add node: **Set**
   - Enable **Include Other Fields**
   - Add numeric fields:
     - `minOrderValueForHighValueTag` = `500`
     - `lifetimeSpendThreshold` = `1000`
   - Connect: **Shopify Trigger → Workflow Configuration**

4) **Add “Extract Customer Data” (Set node)**
   - Add node: **Set**
   - Enable **Include Other Fields**
   - Add fields (use Expressions):
     - `orderTotal` (Number): `{{$json.total_price || 0}}`
     - `orderCount` (Number): `{{$json.customer?.orders_count || 1}}`
     - `lifetimeSpend` (Number): `{{$json.customer?.total_spent || $json.total_price || 0}}`
     - `customerTags` (String): `{{$json.customer?.tags || ''}}`
     - `isHighValue` (Boolean):  
       `{{($json.total_price || 0) >= $('Workflow Configuration').first().json.minOrderValueForHighValueTag}}`
     - `shopifyCustomerId` (String): `{{$json.customer?.id || $json.id}}`
   - Connect: **Workflow Configuration → Extract Customer Data**

5) **Add “Search for Existing Contact” (Zoho CRM)**
   - Add node: **Zoho CRM**
   - Create/select **Zoho OAuth2** credentials:
     - Connect your Zoho account and grant CRM permissions
   - Configure:
     - **Resource:** Contact
     - **Operation:** Get All
     - **Limit:** 2
     - **Options → Fields:** include `Email`
   - Connect: **Extract Customer Data → Search for Existing Contact**
   - (Recommended for a correct rebuild: replace “Get All” with a **Search**/criteria by Email; see notes in section 2.)

6) **Add “Contact Exists?” (IF node)**
   - Add node: **IF**
   - Condition: **String → exists**
     - Left value (Expression): `{{$('Search for Existing Contact').item.json.id}}`
   - Connect: **Search for Existing Contact → Contact Exists?**

7) **Add “Update Existing Contact” (Zoho CRM)**
   - Add node: **Zoho CRM**
   - Configure:
     - **Resource:** Contact
     - **Operation:** Update
     - **Contact ID:** `{{$json.id}}`
     - **Update Fields → Custom Fields:**
       - `Engagement_Score` = `{{$('Extract Customer Data').item.json.lifetimeSpend}}`
       - `Mentions_Counts` = `{{$('Extract Customer Data').item.json.orderCount}}`
   - Connect: **Contact Exists? (true output) → Update Existing Contact**

8) **Add “Create New Contact” (Zoho CRM)**
   - Add node: **Zoho CRM**
   - Configure:
     - **Resource:** Contact
     - **Operation:** Create
     - **Last Name:** `{{$('Extract Customer Data').item.json.fulfillments[0].line_items[0].name}}`
     - Leave additional fields empty (as per provided workflow)
   - Turn on **Always Output Data** for this node.
   - Connect: **Contact Exists? (false output) → Create New Contact**

9) **Activate and test**
   - In Shopify trigger node, ensure webhook is registered (n8n must be reachable).
   - Run a test event from Shopify (create customer/order).
   - Validate Zoho updates/creates.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| # How It Works\nThis workflow automates the synchronization of Shopify customer activity with Zoho CRM, ensuring sales and support teams have up-to-date purchase history and engagement metrics.\n\n# Setup Steps\n## Trigger\nConfigure the Shopify Trigger node with your store's Access Token and set the topic to `customers/create` or `orders/create`.\n\n## Connections\nLink credentials for Shopify (source) and Zoho CRM (destination).\n\n## Configuration\nDefine your high-value customer thresholds (e.g., Min Order Value: 500) in the configuration node to automate customer segmentation.\n\n## Intelligence Sync\nThe system automatically checks if a contact exists in Zoho. If they do, it updates their lifetime spend and order count; if not, it creates a new record. | Canvas sticky note (general workflow explanation) |
| Data-model alignment warning: several expressions (e.g., `total_price`, `fulfillments[0]...`) assume an order-shaped payload; if you keep `customers/create` you should map fields from the customer object (e.g., `last_name`, `email`) and adjust scoring logic accordingly. | Applies to trigger topic and downstream Set/Create expressions |
| Zoho lookup warning: using “Get All” without criteria does not match Shopify customers to Zoho contacts; this can cause wrong updates and duplicates. | Applies to “Search for Existing Contact” and subsequent IF routing |