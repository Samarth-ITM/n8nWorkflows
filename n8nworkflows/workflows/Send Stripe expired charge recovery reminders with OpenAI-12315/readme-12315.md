Send Stripe expired charge recovery reminders with OpenAI

https://n8nworkflows.xyz/workflows/send-stripe-expired-charge-recovery-reminders-with-openai-12315


# Send Stripe expired charge recovery reminders with OpenAI

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:**  
This workflow reacts to Stripe **`charge.expired`** events, extracts customer/payment details, uses OpenAI to draft a polite payment reminder email body, creates a new Stripe invoice/charge for retry, and sends the reminder via Gmail.

**Target use cases:**
- Dunning/recovery for expired payment attempts
- Customer-friendly reminder automation with AI-written messaging
- Lightweight “retry by invoicing” flow for one-off charges (not subscription dunning)

### Logical Blocks
**1.1 Event Reception (Stripe Webhook Trigger)**  
Receives Stripe events and starts the workflow.

**1.2 Workflow & Business Configuration**  
Sets reusable parameters (company name, support email, currency symbol).

**1.3 Data Normalization (Customer + Payment Extraction)**  
Maps Stripe payload fields into normalized variables.

**1.4 AI Message Generation (OpenAI)**  
Generates the reminder email body using a structured instruction prompt.

**1.5 Billing Action (Stripe Invoice/Charge Creation)**  
Creates a new Stripe object to retry collection (configured as “charge create”).

**1.6 Customer Notification (Gmail)**  
Sends the AI-generated reminder email to the customer.

---

## 2. Block-by-Block Analysis

### 1.1 Event Reception (Stripe Webhook Trigger)

**Overview:** Listens for Stripe’s `charge.expired` event and passes the Stripe event payload to downstream nodes.

**Nodes Involved:**  
- **Charge Expired Event**

#### Node: Charge Expired Event
- **Type / Role:** `stripeTrigger` (Webhook-based event trigger)
- **Configuration (interpreted):**
  - Subscribed event: **`charge.expired`**
  - Uses Stripe credentials configured in n8n (not shown in JSON but required)
- **Inputs / Outputs:**
  - **Input:** None (trigger)
  - **Output:** Stripe event payload (commonly includes `data.object` holding the `charge`)
  - Connected to: **Workflow Configuration**
- **Version notes:** `typeVersion: 1` (older Stripe node generation; UI fields may differ vs newer versions)
- **Edge cases / failures:**
  - Stripe webhook signature/endpoint misconfiguration
  - Missing permissions in Stripe account
  - Event payload shape differences if Stripe changes API version
  - Event arrives without expected fields (e.g., missing `billing_details`)

---

### 1.2 Workflow & Business Configuration

**Overview:** Defines constants used across the workflow (branding and formatting).

**Nodes Involved:**  
- **Workflow Configuration**

#### Node: Workflow Configuration
- **Type / Role:** `Set` (creates/overrides fields)
- **Configuration choices:**
  - Adds fields:
    - `companyName` = placeholder (must be replaced)
    - `supportEmail` = placeholder (currently unused in later nodes)
    - `currencySymbol` = `$`
  - **Include Other Fields:** enabled (keeps the Stripe payload and appends these fields)
- **Key expressions / variables:**
  - Static placeholder strings like `<__PLACEHOLDER_VALUE__Your Company Name__>`
- **Inputs / Outputs:**
  - **Input:** Stripe event JSON from trigger
  - **Output:** Original JSON + added configuration fields
  - Connected to: **Extract Customer & Payment Data**
- **Version notes:** `typeVersion: 3.4` (newer Set node schema)
- **Edge cases / failures:**
  - If placeholders are not replaced, emails will contain placeholder text
  - Keeping “include other fields” can inflate payload size (usually fine)

---

### 1.3 Data Normalization (Customer + Payment Extraction)

**Overview:** Extracts the customer name, email, amount, currency, and IDs from the Stripe event payload into normalized top-level fields for later use.

**Nodes Involved:**  
- **Extract Customer & Payment Data**

#### Node: Extract Customer & Payment Data
- **Type / Role:** `Set` (data mapping/normalization)
- **Configuration choices:**
  - **Include Other Fields:** enabled
  - Assignments (computed):
    - `customerName` = `{{$json.data.object.billing_details.name || 'Valued Customer'}}`
    - `customerEmail` = `{{$json.data.object.billing_details.email}}`
    - `chargeAmount` = `{{($json.data.object.amount / 100).toFixed(2)}}`
    - `currency` = `{{$json.data.object.currency.toUpperCase()}}`
    - `chargeId` = `{{$json.data.object.id}}`
    - `customerId` = `{{$json.data.object.customer}}`
- **Key expressions / variables:**
  - Relies on Stripe event structure: `data.object.*`
  - Converts cents to decimal string with 2 decimals
- **Inputs / Outputs:**
  - **Input:** Stripe event JSON + workflow config
  - **Output:** Enriched JSON including normalized fields above
  - Connected to: **Generate Payment Reminder**
- **Version notes:** `typeVersion: 3.4`
- **Edge cases / failures:**
  - `billing_details.email` may be null → downstream Gmail node may fail
  - `data.object.customer` may be null for guest charges → invoice creation will fail
  - `amount` might be missing in unusual events; expression errors possible if path is undefined
  - `chargeAmount` is a **string** (because `toFixed(2)` returns string); later math depends on JS coercion

---

### 1.4 AI Message Generation (OpenAI)

**Overview:** Uses OpenAI (LangChain-based node) to generate a concise, professional reminder email body only (no subject line).

**Nodes Involved:**  
- **Generate Payment Reminder**

#### Node: Generate Payment Reminder
- **Type / Role:** `@n8n/n8n-nodes-langchain.openAi` (LLM text generation)
- **Configuration choices:**
  - **Instructions** (system-like prompt) dynamically includes:
    - `{{ $('Workflow Configuration').first().json.companyName }}`
  - **User/content prompt** includes:
    - Customer name: `{{ $json.customerName }}`
    - Amount with symbol: `{{ $('Workflow Configuration').first().json.currencySymbol }}{{ $json.chargeAmount }}`
    - Currency: `{{ $json.currency }}`
  - Returns “ONLY the email body text”
  - Credentials: OpenAI API (`n8n free OpenAI API credits`)
- **Key expressions / variables:**
  - Cross-node references using `$('Workflow Configuration').first().json...`
- **Inputs / Outputs:**
  - **Input:** Normalized customer/payment fields
  - **Output:** A model response object; downstream node expects an email body field
  - Connected to: **Create New Invoice**
- **Version notes:** `typeVersion: 2` for this OpenAI node
- **Edge cases / failures:**
  - OpenAI auth / quota / rate limits
  - Model output may not match expected property name used later (see Gmail node issue below)
  - Prompt injection risk if customer name contains malicious text (low severity here, but possible)
  - Latency/timeouts

**Important integration note:**  
The Gmail node uses `{{ $('Generate Payment Reminder').first().json.message }}`. Depending on this node’s output schema, the actual text might be in a different field (commonly something like `text`, `content`, or `response`). You should verify the output in an execution and adjust the expression accordingly.

---

### 1.5 Billing Action (Stripe Invoice/Charge Creation)

**Overview:** Creates a new Stripe object to re-attempt payment collection for the same amount/currency.

**Nodes Involved:**  
- **Create New Invoice**

#### Node: Create New Invoice
- **Type / Role:** `Stripe` (API operation)
- **Configuration choices (as configured):**
  - **Resource:** `charge`
  - **Operation:** `create`
  - `amount`: `{{ Math.round($json.chargeAmount * 100) }}`
  - `currency`: `{{ $json.currency.toLowerCase() }}`
  - `customerId`: `{{ $json.customerId }}`
  - `description`: `Payment retry for expired charge {{ $json.chargeId }}`
- **Key expressions / variables:**
  - `chargeAmount` was produced as a string; multiplying by 100 relies on JS coercion
- **Inputs / Outputs:**
  - **Input:** AI output + normalized fields (since prior nodes keep other fields)
  - **Output:** Stripe API response for created charge
  - Connected to: **Send Reminder Email**
- **Version notes:** `typeVersion: 1`
- **Edge cases / failures:**
  - Stripe may not allow creating charges directly depending on account settings / PaymentIntents migration
  - Customer may have no default payment method → charge creation fails
  - Currency mismatch or invalid currency code
  - Amount rounding issues if `chargeAmount` is malformed
  - Authentication / permission errors in Stripe credentials

**Conceptual mismatch to node name:**  
The node is named “Create New Invoice” but configured to create a **Charge** (`resource: charge`). If you truly want an invoice, you’d typically use Stripe **Invoice** + **Invoice Item** (or PaymentIntent flows). As-is, it retries by creating a new charge.

---

### 1.6 Customer Notification (Gmail)

**Overview:** Sends the AI-generated reminder email to the customer’s email address.

**Nodes Involved:**  
- **Send Reminder Email**

#### Node: Send Reminder Email
- **Type / Role:** `Gmail` (send email)
- **Configuration choices:**
  - `sendTo`: `{{ $('Extract Customer & Payment Data').first().json.customerEmail }}`
  - `subject`: `Payment Reminder - {{ $('Workflow Configuration').first().json.companyName }}`
  - `message`: `{{ $('Generate Payment Reminder').first().json.message }}`
  - Sender name: `{{ $('Workflow Configuration').first().json.companyName }}`
- **Inputs / Outputs:**
  - **Input:** Stripe charge creation response (because it is directly connected), but it references prior nodes explicitly via expressions
  - **Output:** Gmail send result (message id, etc.)
- **Version notes:** `typeVersion: 2.1`
- **Edge cases / failures:**
  - Gmail OAuth not configured or token expired
  - `customerEmail` empty/invalid → send fails
  - The OpenAI node output may not contain `.json.message` → email body could be blank or expression error
  - Deliverability issues (SPF/DKIM/DMARC) if sending from a domain mailbox

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Charge Expired Event | stripeTrigger | Entry point: receive Stripe `charge.expired` events | — | Workflow Configuration | ## Overview\nThis workflow automatically handles expired or failed Stripe payments. When a charge expires, it triggers the workflow, drafts a professional AI-generated payment reminder, creates a new Stripe invoice, and optionally sends the reminder to the customer via Email or Slack.\n\n## How it works\n1. Stripe trigger detects expired payment\n2. Set node normalizes customer and payment data\n3. OpenAI node drafts payment reminder message\n4. Stripe node generates a new invoice\n5. Optional Email/Slack node sends message to customer or internal team\n\n## Setup Instructions\n- Stripe: Connect Stripe account in n8n and configure trigger for 'charge.expired'\n- OpenAI: Add API credentials\n- Email/Slack: Configure Email node or Slack notification for sending reminders\n- Optional: Customize AI prompt for brand tone and messaging style\n\n## Stripe\n\nListen for 'charge.expired' or 'invoice.payment_failed' events in Stripe. Make sure the credentials are connected. |
| Workflow Configuration | set | Define company constants (branding, formatting) | Charge Expired Event | Extract Customer & Payment Data | ## Overview\nThis workflow automatically handles expired or failed Stripe payments. When a charge expires, it triggers the workflow, drafts a professional AI-generated payment reminder, creates a new Stripe invoice, and optionally sends the reminder to the customer via Email or Slack.\n\n## How it works\n1. Stripe trigger detects expired payment\n2. Set node normalizes customer and payment data\n3. OpenAI node drafts payment reminder message\n4. Stripe node generates a new invoice\n5. Optional Email/Slack node sends message to customer or internal team\n\n## Setup Instructions\n- Stripe: Connect Stripe account in n8n and configure trigger for 'charge.expired'\n- OpenAI: Add API credentials\n- Email/Slack: Configure Email node or Slack notification for sending reminders\n- Optional: Customize AI prompt for brand tone and messaging style\n\n## Configure \n\nMap the following fields:\n- Customer name\n- Customer email\n- Charge amount\n- Previous invoice ID\n- Currency |
| Extract Customer & Payment Data | set | Extract and normalize Stripe fields | Workflow Configuration | Generate Payment Reminder | ## Overview\nThis workflow automatically handles expired or failed Stripe payments. When a charge expires, it triggers the workflow, drafts a professional AI-generated payment reminder, creates a new Stripe invoice, and optionally sends the reminder to the customer via Email or Slack.\n\n## How it works\n1. Stripe trigger detects expired payment\n2. Set node normalizes customer and payment data\n3. OpenAI node drafts payment reminder message\n4. Stripe node generates a new invoice\n5. Optional Email/Slack node sends message to customer or internal team\n\n## Setup Instructions\n- Stripe: Connect Stripe account in n8n and configure trigger for 'charge.expired'\n- OpenAI: Add API credentials\n- Email/Slack: Configure Email node or Slack notification for sending reminders\n- Optional: Customize AI prompt for brand tone and messaging style\n\n## Configure \n\nMap the following fields:\n- Customer name\n- Customer email\n- Charge amount\n- Previous invoice ID\n- Currency |
| Generate Payment Reminder | @n8n/n8n-nodes-langchain.openAi | Draft reminder email body with OpenAI | Extract Customer & Payment Data | Create New Invoice | ## Overview\nThis workflow automatically handles expired or failed Stripe payments. When a charge expires, it triggers the workflow, drafts a professional AI-generated payment reminder, creates a new Stripe invoice, and optionally sends the reminder to the customer via Email or Slack.\n\n## How it works\n1. Stripe trigger detects expired payment\n2. Set node normalizes customer and payment data\n3. OpenAI node drafts payment reminder message\n4. Stripe node generates a new invoice\n5. Optional Email/Slack node sends message to customer or internal team\n\n## Setup Instructions\n- Stripe: Connect Stripe account in n8n and configure trigger for 'charge.expired'\n- OpenAI: Add API credentials\n- Email/Slack: Configure Email node or Slack notification for sending reminders\n- Optional: Customize AI prompt for brand tone and messaging style\n\n## AI prompt example:\n\"Write a professional, friendly payment reminder for {customer_name} regarding their expired charge of {amount} {currency}. Mention that a new invoice has been generated. Keep it polite and concise.\" |
| Create New Invoice | stripe | Retry billing by creating a Stripe charge (despite node name) | Generate Payment Reminder | Send Reminder Email | ## Overview\nThis workflow automatically handles expired or failed Stripe payments. When a charge expires, it triggers the workflow, drafts a professional AI-generated payment reminder, creates a new Stripe invoice, and optionally sends the reminder to the customer via Email or Slack.\n\n## How it works\n1. Stripe trigger detects expired payment\n2. Set node normalizes customer and payment data\n3. OpenAI node drafts payment reminder message\n4. Stripe node generates a new invoice\n5. Optional Email/Slack node sends message to customer or internal team\n\n## Setup Instructions\n- Stripe: Connect Stripe account in n8n and configure trigger for 'charge.expired'\n- OpenAI: Add API credentials\n- Email/Slack: Configure Email node or Slack notification for sending reminders\n- Optional: Customize AI prompt for brand tone and messaging style\n\n## New Invoice\n\nCreate a new invoice for the customer with the same amount and currency. Use previous invoice metadata to reference subscription or product. |
| Send Reminder Email | gmail | Email the AI reminder to the customer | Create New Invoice | — | ## Overview\nThis workflow automatically handles expired or failed Stripe payments. When a charge expires, it triggers the workflow, drafts a professional AI-generated payment reminder, creates a new Stripe invoice, and optionally sends the reminder to the customer via Email or Slack.\n\n## How it works\n1. Stripe trigger detects expired payment\n2. Set node normalizes customer and payment data\n3. OpenAI node drafts payment reminder message\n4. Stripe node generates a new invoice\n5. Optional Email/Slack node sends message to customer or internal team\n\n## Setup Instructions\n- Stripe: Connect Stripe account in n8n and configure trigger for 'charge.expired'\n- OpenAI: Add API credentials\n- Email/Slack: Configure Email node or Slack notification for sending reminders\n- Optional: Customize AI prompt for brand tone and messaging style\n\n## Send\n\nSend the AI-generated reminder to the customer via email or alert internal staff on Slack. Ensure no sensitive data is exposed. |
| Sticky Note | stickyNote | Documentation/comment | — | — | ## Overview\nThis workflow automatically handles expired or failed Stripe payments. When a charge expires, it triggers the workflow, drafts a professional AI-generated payment reminder, creates a new Stripe invoice, and optionally sends the reminder to the customer via Email or Slack.\n\n## How it works\n1. Stripe trigger detects expired payment\n2. Set node normalizes customer and payment data\n3. OpenAI node drafts payment reminder message\n4. Stripe node generates a new invoice\n5. Optional Email/Slack node sends message to customer or internal team\n\n## Setup Instructions\n- Stripe: Connect Stripe account in n8n and configure trigger for 'charge.expired'\n- OpenAI: Add API credentials\n- Email/Slack: Configure Email node or Slack notification for sending reminders\n- Optional: Customize AI prompt for brand tone and messaging style |
| Sticky Note1 | stickyNote | Documentation/comment | — | — | ## Stripe\n\nListen for 'charge.expired' or 'invoice.payment_failed' events in Stripe. Make sure the credentials are connected. |
| Sticky Note2 | stickyNote | Documentation/comment | — | — | ## Configure \n\nMap the following fields:\n- Customer name\n- Customer email\n- Charge amount\n- Previous invoice ID\n- Currency |
| Sticky Note3 | stickyNote | Documentation/comment | — | — | ## AI prompt example:\n\"Write a professional, friendly payment reminder for {customer_name} regarding their expired charge of {amount} {currency}. Mention that a new invoice has been generated. Keep it polite and concise.\" |
| Sticky Note4 | stickyNote | Documentation/comment | — | — | ## New Invoice\n\nCreate a new invoice for the customer with the same amount and currency. Use previous invoice metadata to reference subscription or product. |
| Sticky Note5 | stickyNote | Documentation/comment | — | — | ## Send\n\nSend the AI-generated reminder to the customer via email or alert internal staff on Slack. Ensure no sensitive data is exposed. |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n  
   - Name: *Automated Stripe Expired Charge Recovery with AI Reminders* (or your preferred name)

2. **Add Trigger Node: “Stripe Trigger”**  
   - Node name: **Charge Expired Event**  
   - Events: select **`charge.expired`**  
   - **Credentials:** connect your **Stripe** account (API key/webhook managed by n8n)  
   - Activate the trigger in Stripe/n8n as required (n8n typically registers a webhook endpoint)

3. **Add a Set node: “Workflow Configuration”**  
   - Add fields:
     - `companyName` (string): your company name
     - `supportEmail` (string): your support address (optional unless you use it later)
     - `currencySymbol` (string): e.g. `$`, `€`, `£`
   - Enable **Include Other Fields**
   - Connect: **Charge Expired Event → Workflow Configuration**

4. **Add a Set node: “Extract Customer & Payment Data”**  
   - Enable **Include Other Fields**
   - Add these fields (expressions):
     - `customerName`:
       - `{{ $json.data.object.billing_details.name || 'Valued Customer' }}`
     - `customerEmail`:
       - `{{ $json.data.object.billing_details.email }}`
     - `chargeAmount`:
       - `{{ ($json.data.object.amount / 100).toFixed(2) }}`
     - `currency`:
       - `{{ $json.data.object.currency.toUpperCase() }}`
     - `chargeId`:
       - `{{ $json.data.object.id }}`
     - `customerId`:
       - `{{ $json.data.object.customer }}`
   - Connect: **Workflow Configuration → Extract Customer & Payment Data**

5. **Add OpenAI (LangChain) node: “Generate Payment Reminder”**  
   - Node type: **OpenAI** (`@n8n/n8n-nodes-langchain.openAi`)  
   - **Credentials:** configure OpenAI API key (or n8n-provided credits if applicable)  
   - Instructions (system-style), paste and adapt (ensure it references your config node):
     - Include `{{ $('Workflow Configuration').first().json.companyName }}`
     - Require: “Return ONLY the email body text”
   - Prompt/content text:
     - `Generate a professional and friendly payment reminder email... Customer name: {{ $json.customerName }}, Amount: {{ $('Workflow Configuration').first().json.currencySymbol }}{{ $json.chargeAmount }} {{ $json.currency }}`
   - Connect: **Extract Customer & Payment Data → Generate Payment Reminder**

6. **Add Stripe node: “Create New Invoice” (as configured: creates a charge)**  
   - Node type: **Stripe**
   - **Credentials:** same Stripe account
   - Resource: **Charge**
   - Operation: **Create**
   - Fields:
     - Amount: `{{ Math.round($json.chargeAmount * 100) }}`
     - Currency: `{{ $json.currency.toLowerCase() }}`
     - Customer ID: `{{ $json.customerId }}`
     - Description: `Payment retry for expired charge {{ $json.chargeId }}`
   - Connect: **Generate Payment Reminder → Create New Invoice**
   - Note: If you truly need an *Invoice*, replace this with Stripe Invoice/Invoice Item nodes (design choice).

7. **Add Gmail node: “Send Reminder Email”**  
   - Node type: **Gmail → Send**
   - **Credentials:** connect Gmail OAuth2 in n8n
   - To: `{{ $('Extract Customer & Payment Data').first().json.customerEmail }}`
   - Subject: `Payment Reminder - {{ $('Workflow Configuration').first().json.companyName }}`
   - Sender Name: `{{ $('Workflow Configuration').first().json.companyName }}`
   - Message/body: **verify OpenAI output field**:
     - As in workflow: `{{ $('Generate Payment Reminder').first().json.message }}`
     - If empty, run once and switch to the actual returned field (commonly `text`/`content`)
   - Connect: **Create New Invoice → Send Reminder Email**

8. **Test end-to-end**
   - Use Stripe test mode and simulate a `charge.expired` event (or trigger via Stripe CLI)
   - Confirm:
     - customer email exists
     - OpenAI node returns body text in the field your Gmail node references
     - Stripe charge/invoice creation succeeds

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Listen for `charge.expired` or `invoice.payment_failed` events in Stripe; ensure credentials are connected. | Stripe event selection guidance (from sticky note) |
| Map customer name/email/amount/currency and reference IDs from the Stripe payload. | Data normalization guidance (from sticky note) |
| AI prompt suggestion: “Write a professional, friendly payment reminder…” | Prompt example (from sticky note) |
| Ensure no sensitive data is exposed when sending reminders; optionally notify internal staff on Slack. | Delivery/security guidance (from sticky note) |
| This workflow claims “Create a new invoice”, but the configured Stripe operation actually creates a **charge**. Consider adjusting to Invoice/PaymentIntent flows if required. | Implementation consistency note (analysis) |