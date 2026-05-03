Send welcome WhatsApp messages to new HubSpot contacts with Beex templates

https://n8nworkflows.xyz/workflows/send-welcome-whatsapp-messages-to-new-hubspot-contacts-with-beex-templates-11712


# Send welcome WhatsApp messages to new HubSpot contacts with Beex templates

## 1. Workflow Overview

**Purpose:**  
This workflow listens for **new HubSpot contact creation events** (delivered to n8n via a webhook), fetches the contact’s key properties from HubSpot, validates that the contact has both an **email** and a **phone number**, then sends a **WhatsApp template message** using **Beex** (community node `n8n-nodes-beex`).

**Target use cases:**
- Automatically sending welcome/first-contact WhatsApp messages when a contact is created in HubSpot.
- Standardizing phone formatting (country code + number) before submitting to Beex templates.
- Ensuring minimum contact quality (email + phone) before messaging.

### 1.1 Input Reception (HubSpot → n8n)
Receives a HubSpot event payload via an n8n webhook (expected to include a contact/object ID).

### 1.2 HubSpot Enrichment
Uses the received contact ID to retrieve contact properties from HubSpot.

### 1.3 Validation Gate
Filters out contacts missing required fields (email and calculated phone number).

### 1.4 Normalization / Mapping
Derives `country_code`, `phone_number`, `template_name`, and template variables for Beex.

### 1.5 WhatsApp Template Send (Beex)
Posts a template send request to Beex with the mapped fields.

---

## 2. Block-by-Block Analysis

### Block 1 — Input Reception (HubSpot webhook)
**Overview:** Receives an HTTP POST call from HubSpot when a contact is created. The payload is later used to extract the contact ID.  
**Nodes involved:** `Webhook`

#### Node: Webhook
- **Type / Role:** `n8n-nodes-base.webhook` — entry trigger that starts the workflow on inbound HTTP calls.
- **Key configuration (interpreted):**
  - **HTTP Method:** `POST`
  - **Path:** `3757267f-2176-4a9c-b564-9615d4574262` (also the webhookId)
  - **Expected payload usage:** downstream node expects `objectId` at `body[0].objectId`
- **Key expressions / variables:** none in this node.
- **Connections:**
  - **Output →** `Get Contact`
- **Version-specific notes:** typeVersion `2.1` (webhook node behavior may differ slightly across n8n versions; path and method remain stable).
- **Edge cases / failures:**
  - HubSpot may send a payload that does **not** match `body[0].objectId` (different subscription format), causing downstream HubSpot “Get Contact” to fail.
  - If webhook is not publicly reachable (self-hosting/firewall), HubSpot delivery will fail.
  - If you switch between test/production webhook URLs in n8n, HubSpot must be configured with the correct one.

---

### Block 2 — HubSpot Enrichment (fetch contact properties)
**Overview:** Uses the contact ID from the webhook payload to fetch key properties (email, phone, WhatsApp phone, first name).  
**Nodes involved:** `Get Contact`

#### Node: Get Contact
- **Type / Role:** `n8n-nodes-base.hubspot` — retrieves a HubSpot contact record by ID.
- **Key configuration (interpreted):**
  - **Authentication:** App Token (`authentication: appToken`)
  - **Operation:** `get` (contact by ID)
  - **Contact ID expression:** `={{ $json.body[0].objectId }}`
  - **Requested properties:** `email`, `phone`, `hs_whatsapp_phone_number`, `firstname`
    - Configured as “valueOnly” style extraction in the node options.
- **Connections:**
  - **Input ←** `Webhook`
  - **Output →** `Validate Contact`
- **Credentials:**
  - `hubspotAppToken` must have sufficient scopes/permissions to read contacts.
- **Version-specific notes:** typeVersion `2.2`.
- **Edge cases / failures:**
  - Invalid/missing `objectId` → HubSpot API call fails.
  - App token revoked/expired/insufficient scopes → 401/403 errors.
  - Contact ID exists but properties are empty → validation will block later.
  - HubSpot property naming mismatch: later nodes validate `hs_calculated_phone_number`, but this node requests `phone`, `hs_whatsapp_phone_number` (see validation block notes).

---

### Block 3 — Validation Gate (email + phone required)
**Overview:** Ensures the contact has a usable email and phone number before attempting WhatsApp send. Prevents unnecessary Beex calls.  
**Nodes involved:** `Validate Contact`

#### Node: Validate Contact
- **Type / Role:** `n8n-nodes-base.filter` — conditional pass-through.
- **Key configuration (interpreted):**
  - Uses an **AND** combinator across conditions.
  - Conditions (as configured):
    1. `email` **exists**: `={{ $json.properties.email.value }}`
    2. `hs_calculated_phone_number` **exists**: `={{ $json.properties.hs_calculated_phone_number.value }}`
    3. `email` **not empty**
    4. `hs_calculated_phone_number` **not empty**
- **Connections:**
  - **Input ←** `Get Contact`
  - **Output (true path) →** `Set Fields`
  - **Output (false path) →** nothing (items are dropped)
- **Version-specific notes:** typeVersion `2.2`.
- **Edge cases / failures:**
  - **High-risk mismatch:** This node checks `hs_calculated_phone_number`, but the HubSpot “Get Contact” node does **not** request that property. If HubSpot doesn’t return it by default, this filter will fail and drop all items.
    - Fix options: request `hs_calculated_phone_number` in **Get Contact**, or validate against `phone`/`hs_whatsapp_phone_number` instead.
  - If HubSpot returns properties in a different structure (e.g., already flattened), expressions like `$json.properties.email.value` may be undefined.

---

### Block 4 — Normalization / Mapping for Beex template
**Overview:** Derives Beex-required fields (country code, phone number, template name, and template variables) from HubSpot properties.  
**Nodes involved:** `Set Fields`

#### Node: Set Fields
- **Type / Role:** `n8n-nodes-base.set` — maps/transforms fields for downstream API call.
- **Key configuration (interpreted):** creates the following fields:
  - `country_code`:
    - Expression:  
      `={{ $json.properties.hs_calculated_phone_number.value.slice(1,3) }}`
    - Assumes phone is in an international format like `+33XXXXXXXXX` and that the country code is exactly **2 digits** after `+`.
  - `phone_number`:
    - Expression:  
      `={{ $json.properties.hs_calculated_phone_number.value.slice(3) }}`
    - Removes `+` and 2-digit country code.
  - `template_name`: hardcoded to `n8n_beex`
  - `associated_values`:
    - Expression: `=["{{ $json.properties.firstname.value }}"]`
    - Produces a JSON-like array string containing the firstname (as written, it’s a string that looks like an array; whether Beex expects a true array vs string depends on the community node implementation).
- **Connections:**
  - **Input ←** `Validate Contact`
  - **Output →** `Send Template`
- **Version-specific notes:** typeVersion `3.4`.
- **Edge cases / failures:**
  - **Country code slicing is brittle:**
    - Many country codes are 1–3 digits; `.slice(1,3)` only works reliably for 2-digit codes.
    - If the number isn’t prefixed with `+`, slicing is wrong.
  - If `hs_calculated_phone_number` is missing, expressions throw/resolve to empty → Beex request may fail.
  - `associated_values` may need to be a real array (not a string). If Beex node expects JSON, you may need an expression that returns an actual array: `={{ [$json.properties.firstname.value] }}`.

---

### Block 5 — WhatsApp Template Send (Beex)
**Overview:** Sends a WhatsApp template message through Beex using the mapped fields.  
**Nodes involved:** `Send Template`

#### Node: Send Template
- **Type / Role:** `n8n-nodes-beex.beex` — community Beex API node; sends a template message.
- **Key configuration (interpreted):**
  - **Resource:** `templates`
  - **Operation:** `post`
  - **Queue ID:** `38` (Beex-side routing/queue selection)
  - **Parameters from previous node:**
    - `code_country`: `={{ $json.country_code }}`
    - `phone_number`: `={{ $json.phone_number }}`
    - `template_name`: `={{ $json.template_name }}`
    - `template_values`: `={{ $json.associated_values }}`
- **Connections:**
  - **Input ←** `Set Fields`
  - **Output →** none (end)
- **Credentials:**
  - `beexApi` credential (typically a bearer token) must be configured and authorized to send templates.
- **Version-specific notes:** typeVersion `1` (community node; behavior depends on installed package version).
- **Edge cases / failures:**
  - Invalid token / missing permissions → authentication errors.
  - Wrong template name or template variables count/type mismatch → Beex API validation errors.
  - Phone formatting invalid for Beex/WhatsApp → message rejected.
  - Queue ID incorrect → routing failure.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Webhook | n8n-nodes-base.webhook | Receives HubSpot contact-created event | — | Get Contact | ## Trigger Node (HubSpot Webhook + Get Data)\n- Link the webhook **URL** on the CRM HubSpot\n- Get HubSpot contact properties by ID\n- Validate contact (email and phone number) |
| Get Contact | n8n-nodes-base.hubspot | Fetch contact details by ID | Webhook | Validate Contact | ## Trigger Node (HubSpot Webhook + Get Data)\n- Link the webhook **URL** on the CRM HubSpot\n- Get HubSpot contact properties by ID\n- Validate contact (email and phone number) |
| Validate Contact | n8n-nodes-base.filter | Enforce required fields (email + phone) | Get Contact | Set Fields | ## Trigger Node (HubSpot Webhook + Get Data)\n- Link the webhook **URL** on the CRM HubSpot\n- Get HubSpot contact properties by ID\n- Validate contact (email and phone number) |
| Set Fields | n8n-nodes-base.set | Transform HubSpot properties to Beex template inputs | Validate Contact | Send Template | ## Set Fields\n- The contact fields are extracted and transformed as needed for use in the template submission node. |
| Send Template | n8n-nodes-beex.beex | Send WhatsApp template via Beex | Set Fields | — | ## Warning\n- Configure template in Beex |

Sticky note present in workflow canvas but not tied to a single node (general):  
- **Beex First Contact for HubSpot (WhatsApp)** note with requirements + video link applies to the overall workflow.

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.
2. **Add node: Webhook**
   - Node type: **Webhook**
   - Method: **POST**
   - Path: choose a unique path (n8n will generate one if desired)
   - Activate “Production URL” usage according to your deployment.
3. **Configure HubSpot to call the webhook**
   - In a **HubSpot custom app**, create a subscription for **Contact creation** events.
   - Set the subscription target URL to the n8n webhook **Production URL**.
   - Ensure HubSpot payload includes the contact ID in a structure you can reference (the provided workflow expects `body[0].objectId`).
4. **Add node: HubSpot → Get Contact**
   - Node type: **HubSpot**
   - Authentication: **App Token**
   - Operation: **Get Contact**
   - Contact ID expression: `={{ $json.body[0].objectId }}`
   - Properties to request: at least `email`, `firstname`, and the phone property you will use.
     - To match the existing workflow logic, also request: **`hs_calculated_phone_number`** (important) or adjust later nodes.
   - Add HubSpot credentials (App Token) with read access to Contacts.
5. **Connect:** `Webhook → Get Contact`
6. **Add node: Filter → “Validate Contact”**
   - Node type: **Filter**
   - Combinator: **AND**
   - Conditions (example matching the workflow):
     - `{{$json.properties.email.value}}` **exists**
     - `{{$json.properties.hs_calculated_phone_number.value}}` **exists**
     - `{{$json.properties.email.value}}` **not empty**
     - `{{$json.properties.hs_calculated_phone_number.value}}` **not empty**
7. **Connect:** `Get Contact → Validate Contact`
8. **Add node: Set → “Set Fields”**
   - Node type: **Set**
   - Add fields:
     - `country_code` (string): `={{ $json.properties.hs_calculated_phone_number.value.slice(1,3) }}`
     - `phone_number` (string): `={{ $json.properties.hs_calculated_phone_number.value.slice(3) }}`
     - `template_name` (string): `n8n_beex`
     - `associated_values` (string): `=["{{ $json.properties.firstname.value }}"]`
       - If Beex expects a real array, use instead: `={{ [$json.properties.firstname.value] }}`
9. **Connect:** `Validate Contact → Set Fields`
10. **Install and enable the community node**
    - Install `n8n-nodes-beex` in your n8n instance (Community Nodes).
11. **Add node: Beex → “Send Template”**
    - Node type: **Beex**
    - Resource: **templates**
    - Operation: **post**
    - Queue ID: `38` (or your Beex queue)
    - Map fields:
      - `code_country`: `={{ $json.country_code }}`
      - `phone_number`: `={{ $json.phone_number }}`
      - `template_name`: `={{ $json.template_name }}`
      - `template_values`: `={{ $json.associated_values }}`
    - Configure Beex credentials (bearer token / API token depending on node credential type).
12. **Connect:** `Set Fields → Send Template`
13. **Test end-to-end**
    - Trigger a HubSpot contact creation event.
    - Verify HubSpot payload mapping (`objectId` path).
    - Confirm validation passes and Beex accepts the template payload.
14. **Activate the workflow** in n8n once confirmed.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| This workflow requires the community node `n8n-nodes-beex`. | Applies to the Beex “Send Template” node |
| Watch This Tutorial: @[youtube](oX6UxcBDlI0) | Video referenced in the workflow sticky note |
| Ensure HubSpot webhook URL is linked in HubSpot CRM custom app and sends contact creation events. | HubSpot → n8n integration requirement |
| Adjust `country_code` and `phone_number` extraction according to your region/phone formats. | Set Fields node logic is format-dependent |
| Configure template in Beex (template name `n8n_beex` used by default). | Beex template must exist and match variables |

Disclaimer (as provided): Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.