Send 10-day post-purchase WhatsApp offers with Odoo, OpenAI and Evolution API

https://n8nworkflows.xyz/workflows/send-10-day-post-purchase-whatsapp-offers-with-odoo--openai-and-evolution-api-14829


# Send 10-day post-purchase WhatsApp offers with Odoo, OpenAI and Evolution API

# 1. Workflow Overview

This workflow automatically identifies customers in Odoo who made a purchase **10 days earlier**, generates a **personalized Arabic WhatsApp offer** using OpenAI, and sends the offer through the **Evolution API** with a **1-minute delay between messages** to reduce spam or ban risk.

Its main use case is **post-purchase retargeting** for e-commerce or retail brands using Odoo as the source of invoice/customer data and WhatsApp as the outreach channel.

## 1.1 Scheduled Trigger and Target Invoice Retrieval
The workflow starts every day at **10:01 AM**. It queries Odoo invoices and filters them by date so only invoices from the intended time window are retrieved.

## 1.2 Customer Enrichment
For each matching invoice, the workflow retrieves the linked customer record from Odoo in order to access contact information such as name, mobile number, or phone number.

## 1.3 Sequential Processing and AI Copy Generation
Customers are processed **one at a time** using a batch loop. For each customer, OpenAI generates a tailored WhatsApp message in **Egyptian colloquial Arabic**, following detailed brand and copywriting constraints.

## 1.4 WhatsApp Delivery and Anti-Ban Delay
The generated message is sent via an HTTP request to the Evolution API. After each send attempt, the workflow waits **1 minute** before looping back to process the next customer.

---

# 2. Block-by-Block Analysis

## Block 1 — Scheduled Trigger and Invoice Retrieval

### Overview
This block runs the workflow daily and fetches invoices from Odoo that match the target age criteria. It forms the data entry point for the entire automation.

### Nodes Involved
- Daily Check (10:01 AM)
- Fetch 10-Day Old Invoices

### Node Details

#### 1. Daily Check (10:01 AM)
- **Type and technical role:** `n8n-nodes-base.scheduleTrigger`  
  Starts the workflow automatically on a daily schedule.
- **Configuration choices:**
  - Configured to trigger every day at **10:01 AM**.
- **Key expressions or variables used:**
  - None.
- **Input and output connections:**
  - No input; this is an entry-point node.
  - Outputs to **Fetch 10-Day Old Invoices**.
- **Version-specific requirements:**
  - Uses node type version **1.3**.
- **Edge cases or potential failure types:**
  - Timezone differences between the n8n instance and business timezone can cause records to be selected on the wrong day.
  - If the workflow is disabled, no execution occurs.
- **Sub-workflow reference:**
  - None.

#### 2. Fetch 10-Day Old Invoices
- **Type and technical role:** `n8n-nodes-base.odoo`  
  Queries Odoo custom resources, here used to retrieve invoice records from `account.move`.
- **Configuration choices:**
  - Resource set to **custom**.
  - Operation set to **getAll**.
  - `returnAll` enabled, meaning all matching invoice records are returned.
  - Custom resource: `account.move`.
  - Filter includes:
    - `date = {{$today.minus({ days: 9 }).toFormat('yyyy-MM-dd') }}`
    - `date >= 2026-04-01`
- **Key expressions or variables used:**
  - `{{$today.minus({ days: 9 }).toFormat('yyyy-MM-dd') }}`
- **Input and output connections:**
  - Input from **Daily Check (10:01 AM)**.
  - Output to **Retrieve Customer Contact**.
- **Version-specific requirements:**
  - Uses node type version **1**.
- **Edge cases or potential failure types:**
  - The expression uses **9 days**, while the workflow title and comments describe **10-day post-purchase** logic. This may be intentional depending on inclusive date handling, but it is a likely place to verify.
  - Odoo date filtering behavior may depend on server timezone and field type.
  - If many invoices match, `returnAll` may produce a large dataset.
  - Odoo auth, permission, or custom model access issues may prevent retrieval.
  - Invoices without `partner_id` will break downstream contact lookup.
- **Sub-workflow reference:**
  - None.

---

## Block 2 — Customer Enrichment

### Overview
This block retrieves the full customer record associated with each invoice. It supplies the name and phone information required for personalization and WhatsApp delivery.

### Nodes Involved
- Retrieve Customer Contact

### Node Details

#### 3. Retrieve Customer Contact
- **Type and technical role:** `n8n-nodes-base.odoo`  
  Retrieves a single Odoo record from the customer model `res.partner`.
- **Configuration choices:**
  - Resource set to **custom**.
  - Operation set to **get**.
  - Custom resource: `res.partner`
  - Record ID is taken from the first element of the invoice’s `partner_id` array.
- **Key expressions or variables used:**
  - `={{ $json.partner_id[0] }}`
- **Input and output connections:**
  - Input from **Fetch 10-Day Old Invoices**.
  - Output to **Process Customers (One by One)**.
- **Version-specific requirements:**
  - Uses node type version **1**.
- **Edge cases or potential failure types:**
  - If `partner_id` is null, empty, or not an array, the expression fails.
  - If the partner record has no `mobile` and no `phone`, the WhatsApp send step will generate an empty or invalid number.
  - Odoo permission issues on `res.partner` can stop execution.
- **Sub-workflow reference:**
  - None.

---

## Block 3 — Sequential Processing and AI Message Generation

### Overview
This block ensures customers are handled one at a time, then generates a personalized marketing message using OpenAI. The loop structure is central to rate limiting and personalization.

### Nodes Involved
- Process Customers (One by One)
- Generate AI Message

### Node Details

#### 4. Process Customers (One by One)
- **Type and technical role:** `n8n-nodes-base.splitInBatches`  
  Iterates through incoming customer records one by one.
- **Configuration choices:**
  - Uses default batching behavior.
  - In this structure:
    - The **loop output** sends one item at a time to the AI node.
    - After send + wait, the flow returns to this node to request the next item.
- **Key expressions or variables used:**
  - None directly in parameters.
- **Input and output connections:**
  - Input from **Retrieve Customer Contact**.
  - Second output goes to **Generate AI Message**.
  - Receives a loop-back input from **Wait 1 Mins (Anti-Ban)**.
- **Version-specific requirements:**
  - Uses node type version **3**.
- **Edge cases or potential failure types:**
  - If no customers are retrieved, this node will not emit loop items.
  - Incorrect loop wiring can cause infinite loops or skipped items; here the wiring is valid for sequential pacing.
- **Sub-workflow reference:**
  - None.

#### 5. Generate AI Message
- **Type and technical role:** `@n8n/n8n-nodes-langchain.openAi`  
  Sends a prompt to OpenAI to generate a final Arabic WhatsApp message.
- **Configuration choices:**
  - Model: **gpt-4o-mini**
  - Uses structured response messages with:
    - A **system prompt** defining brand voice, offer details, language style, CTA requirements, emoji usage, and forbidden phrasing.
    - A **user prompt** instructing the model to:
      - write the discount message,
      - use only the customer’s first name,
      - transliterate English names naturally into Arabic,
      - default to `عميلنا العزيز` if no name is present.
- **Key expressions or variables used:**
  - `{{ $json.name || 'عميلنا العزيز' }}`
- **Input and output connections:**
  - Input from **Process Customers (One by One)**.
  - Output to **Send WhatsApp Discount**.
- **Version-specific requirements:**
  - Uses node type version **2.1**.
  - Requires the LangChain/OpenAI node package available in the n8n instance.
- **Edge cases or potential failure types:**
  - OpenAI credential issues or quota/rate-limit failures.
  - The prompt asks for “first name only”, but no preprocessing node actually extracts the first name; the model is relied upon to infer it from `name`.
  - Arabic transliteration quality may vary.
  - Response structure is assumed downstream as `$json.output[0].content[0].text`; if the node output format changes by version or API mode, the HTTP node will fail.
- **Sub-workflow reference:**
  - None.

---

## Block 4 — WhatsApp Sending and Anti-Ban Loop

### Overview
This block sends the generated message to WhatsApp through Evolution API, then pauses for one minute before continuing with the next customer. It is designed to reduce messaging frequency and operational risk.

### Nodes Involved
- Send WhatsApp Discount
- Wait 1 Mins (Anti-Ban)

### Node Details

#### 6. Send WhatsApp Discount
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Sends the final WhatsApp message payload to an external HTTP API endpoint.
- **Configuration choices:**
  - Method: **POST**
  - URL: placeholder `YOUR_EVOLUTION_API_URL`
  - Body sent as **JSON**
  - Header includes:
    - `apikey: YOUR_EVOLUTION_API_KEY`
  - Error handling:
    - `onError: continueRegularOutput`, so execution continues even if sending fails.
  - Request body:
    - `number`: built from customer `mobile` or `phone`
    - `text`: extracted from OpenAI output
- **Key expressions or variables used:**
  - Number formatting logic:
    - Uses `$('Retrieve Customer Contact').item.json.mobile || $('Retrieve Customer Contact').item.json.phone || ''`
    - Removes non-digits with `.replace(/\D/g, '')`
    - If number starts with `0`, replaces prefix with `20`
    - If number starts with `1`, prepends `20`
  - Message text:
    - `JSON.stringify($json.output[0].content[0].text || "")`
- **Input and output connections:**
  - Input from **Generate AI Message**.
  - Output to **Wait 1 Mins (Anti-Ban)**.
- **Version-specific requirements:**
  - Uses node type version **4.3**.
- **Edge cases or potential failure types:**
  - Placeholder URL and API key must be replaced.
  - The phone normalization logic assumes Egyptian numbers and may produce invalid results for other locales.
  - If both mobile and phone are missing, the API may receive an empty number.
  - If OpenAI output is missing or structurally different, `text` may be empty.
  - Because errors continue on regular output, failed sends may appear as processed unless separately logged.
  - Endpoint path may need to match the exact Evolution API message endpoint; only a base placeholder is shown here.
- **Sub-workflow reference:**
  - None.

#### 7. Wait 1 Mins (Anti-Ban)
- **Type and technical role:** `n8n-nodes-base.wait`  
  Inserts a fixed delay before continuing to the next customer.
- **Configuration choices:**
  - Wait duration: **1 minute**
- **Key expressions or variables used:**
  - None.
- **Input and output connections:**
  - Input from **Send WhatsApp Discount**.
  - Output loops back to **Process Customers (One by One)**.
- **Version-specific requirements:**
  - Uses node type version **1.1**.
- **Edge cases or potential failure types:**
  - Large customer lists will significantly extend total execution time.
  - Wait nodes can be affected by execution mode and persistence settings in self-hosted n8n.
- **Sub-workflow reference:**
  - None.

---

## Non-Executable Documentation Nodes

These nodes do not participate in execution but provide visual context in the canvas.

### 8. Sticky Note
- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Canvas documentation note describing the workflow purpose and setup requirements.
- **Configuration choices:**
  - Contains operational explanation and setup checklist for Odoo, OpenAI, and WhatsApp.
- **Connections:** None.
- **Edge cases:** None.

### 9. Sticky Note 1
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:**
  - Documents the customer fetch stage.
- **Connections:** None.
- **Edge cases:** None.

### 10. Sticky Note 2
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:**
  - Documents the AI and loop stage.
- **Connections:** None.
- **Edge cases:** None.

### 11. Sticky Note 3
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:**
  - Documents the sending and anti-ban stage.
- **Connections:** None.
- **Edge cases:** None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Daily Check (10:01 AM) | Schedule Trigger | Starts the workflow daily at 10:01 AM |  | Fetch 10-Day Old Invoices | ### How it works<br>This workflow automatically detects customers who made a purchase exactly 10 days ago, generates a highly personalized, natural-sounding Arabic promotional message using OpenAI, and sends it directly via WhatsApp. It includes an anti-ban loop to pace the messages.<br><br>### Setup steps<br>1. **Odoo:** Connect your credentials to fetch Invoices and Contacts.<br>2. **OpenAI:** Add your API Key to the AI Node to generate the custom copywriting.<br>3. **WhatsApp:** Update the HTTP Request node with your Evolution API credentials (or switch to your preferred WhatsApp provider node).<br>## 1. Fetch Target Customers<br>Triggers daily at 10:01 AM, searches Odoo for invoices that are exactly 10 days old, and retrieves the customer's name and phone number. |
| Fetch 10-Day Old Invoices | Odoo | Retrieves matching invoice records from Odoo `account.move` | Daily Check (10:01 AM) | Retrieve Customer Contact | ### How it works<br>This workflow automatically detects customers who made a purchase exactly 10 days ago, generates a highly personalized, natural-sounding Arabic promotional message using OpenAI, and sends it directly via WhatsApp. It includes an anti-ban loop to pace the messages.<br><br>### Setup steps<br>1. **Odoo:** Connect your credentials to fetch Invoices and Contacts.<br>2. **OpenAI:** Add your API Key to the AI Node to generate the custom copywriting.<br>3. **WhatsApp:** Update the HTTP Request node with your Evolution API credentials (or switch to your preferred WhatsApp provider node).<br>## 1. Fetch Target Customers<br>Triggers daily at 10:01 AM, searches Odoo for invoices that are exactly 10 days old, and retrieves the customer's name and phone number. |
| Retrieve Customer Contact | Odoo | Fetches the customer partner record linked to each invoice | Fetch 10-Day Old Invoices | Process Customers (One by One) | ### How it works<br>This workflow automatically detects customers who made a purchase exactly 10 days ago, generates a highly personalized, natural-sounding Arabic promotional message using OpenAI, and sends it directly via WhatsApp. It includes an anti-ban loop to pace the messages.<br><br>### Setup steps<br>1. **Odoo:** Connect your credentials to fetch Invoices and Contacts.<br>2. **OpenAI:** Add your API Key to the AI Node to generate the custom copywriting.<br>3. **WhatsApp:** Update the HTTP Request node with your Evolution API credentials (or switch to your preferred WhatsApp provider node).<br>## 1. Fetch Target Customers<br>Triggers daily at 10:01 AM, searches Odoo for invoices that are exactly 10 days old, and retrieves the customer's name and phone number. |
| Process Customers (One by One) | Split In Batches | Iterates through customers sequentially | Retrieve Customer Contact, Wait 1 Mins (Anti-Ban) | Generate AI Message | ## 2. AI Copywriting & Loop<br>Processes the customers one by one. The AI Agent takes the customer's name and generates a personalized, colloquial Egyptian Arabic message offering an extra 10% discount. |
| Generate AI Message | OpenAI (LangChain) | Generates personalized Arabic WhatsApp copy | Process Customers (One by One) | Send WhatsApp Discount | ## 2. AI Copywriting & Loop<br>Processes the customers one by one. The AI Agent takes the customer's name and generates a personalized, colloquial Egyptian Arabic message offering an extra 10% discount. |
| Send WhatsApp Discount | HTTP Request | Sends the generated message to Evolution API | Generate AI Message | Wait 1 Mins (Anti-Ban) | ## 3. WhatsApp Sending & Anti-Ban<br>Sends the AI-generated message via Evolution API to the customer's WhatsApp, then pauses for 1 minute before sending the next message to prevent spam bans. |
| Wait 1 Mins (Anti-Ban) | Wait | Delays the next iteration by 1 minute | Send WhatsApp Discount | Process Customers (One by One) | ## 3. WhatsApp Sending & Anti-Ban<br>Sends the AI-generated message via Evolution API to the customer's WhatsApp, then pauses for 1 minute before sending the next message to prevent spam bans. |
| Sticky Note | Sticky Note | Canvas documentation |  |  | ### How it works<br>This workflow automatically detects customers who made a purchase exactly 10 days ago, generates a highly personalized, natural-sounding Arabic promotional message using OpenAI, and sends it directly via WhatsApp. It includes an anti-ban loop to pace the messages.<br><br>### Setup steps<br>1. **Odoo:** Connect your credentials to fetch Invoices and Contacts.<br>2. **OpenAI:** Add your API Key to the AI Node to generate the custom copywriting.<br>3. **WhatsApp:** Update the HTTP Request node with your Evolution API credentials (or switch to your preferred WhatsApp provider node). |
| Sticky Note 1 | Sticky Note | Canvas documentation for fetch stage |  |  | ## 1. Fetch Target Customers<br>Triggers daily at 10:01 AM, searches Odoo for invoices that are exactly 10 days old, and retrieves the customer's name and phone number. |
| Sticky Note 2 | Sticky Note | Canvas documentation for AI stage |  |  | ## 2. AI Copywriting & Loop<br>Processes the customers one by one. The AI Agent takes the customer's name and generates a personalized, colloquial Egyptian Arabic message offering an extra 10% discount. |
| Sticky Note 3 | Sticky Note | Canvas documentation for WhatsApp stage |  |  | ## 3. WhatsApp Sending & Anti-Ban<br>Sends the AI-generated message via Evolution API to the customer's WhatsApp, then pauses for 1 minute before sending the next message to prevent spam bans. |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it something like: **AI-Powered WhatsApp Retargeting: Smart 10-Day Post-Purchase Offers**.

2. **Add a Schedule Trigger node**
   - Node type: **Schedule Trigger**
   - Name: **Daily Check (10:01 AM)**
   - Configure one daily interval:
     - Hour: **10**
     - Minute: **1**
   - Ensure the workflow timezone matches your business timezone.

3. **Add an Odoo node to fetch invoices**
   - Node type: **Odoo**
   - Name: **Fetch 10-Day Old Invoices**
   - Credential: connect valid **Odoo credentials**
   - Set:
     - Resource: **Custom**
     - Operation: **Get All**
     - Custom Resource: `account.move`
     - Return All: **true**
   - Add filter conditions:
     - `date` equals `{{ $today.minus({ days: 9 }).toFormat('yyyy-MM-dd') }}`
     - `date` greater or equal to `2026-04-01`
   - Connect **Daily Check (10:01 AM)** → **Fetch 10-Day Old Invoices**
   - Important: verify whether your business logic truly requires **9 days** or **10 days**. If you want exact 10-day offset by plain interpretation, test whether this should instead be `days: 10`.

4. **Add an Odoo node to fetch customer details**
   - Node type: **Odoo**
   - Name: **Retrieve Customer Contact**
   - Credential: reuse the same Odoo credential
   - Set:
     - Resource: **Custom**
     - Operation: **Get**
     - Custom Resource: `res.partner`
     - Custom Resource ID: `{{ $json.partner_id[0] }}`
   - Connect **Fetch 10-Day Old Invoices** → **Retrieve Customer Contact**

5. **Add a Split In Batches node**
   - Node type: **Split In Batches**
   - Name: **Process Customers (One by One)**
   - Leave default batch settings so items are processed sequentially.
   - Connect **Retrieve Customer Contact** → **Process Customers (One by One)**

6. **Add the OpenAI message generation node**
   - Node type: **OpenAI** using the LangChain/OpenAI integration
   - Name: **Generate AI Message**
   - Credential: connect valid **OpenAI API credentials**
   - Model:
     - Select **gpt-4o-mini**
   - In the response/messages configuration, add:
     - A **system message** containing the marketing and formatting instructions
     - A **user message** containing the customer-specific request and name expression
   - Use this logic in the user message:
     - `Customer Name: {{ $json.name || 'عميلنا العزيز' }}`
   - Preserve the intent of the original prompt:
     - Egyptian colloquial Arabic
     - extra 10% discount
     - mention it is an additional discount
     - no legal disclaimers
     - CTA to contact the page via Facebook/Instagram
     - output only the final text
   - Connect the **second/main loop output** of **Process Customers (One by One)** → **Generate AI Message**

7. **Add an HTTP Request node for WhatsApp sending**
   - Node type: **HTTP Request**
   - Name: **Send WhatsApp Discount**
   - Set:
     - Method: **POST**
     - URL: your actual **Evolution API endpoint**
     - Send Headers: **true**
     - Send Body: **true**
     - Body Type / Specify Body: **JSON**
     - Error handling: set to **Continue Regular Output**
   - Add header:
     - `apikey` = your actual Evolution API key
   - Use a JSON body equivalent to:
     - `number`: derived from `mobile` or `phone`
     - `text`: the generated AI message
   - Recreate the phone-cleaning logic:
     - take `mobile`, otherwise `phone`
     - remove all non-digit characters
     - if it starts with `0`, replace leading `0` with `20`
     - if it starts with `1`, prepend `20`
   - Recreate the text extraction logic from the OpenAI node output:
     - use the returned message text field corresponding to your n8n/OpenAI node version
   - Connect **Generate AI Message** → **Send WhatsApp Discount**

8. **Add a Wait node**
   - Node type: **Wait**
   - Name: **Wait 1 Mins (Anti-Ban)**
   - Configure:
     - Unit: **Minutes**
     - Amount: **1**
   - Connect **Send WhatsApp Discount** → **Wait 1 Mins (Anti-Ban)**

9. **Close the loop**
   - Connect **Wait 1 Mins (Anti-Ban)** → **Process Customers (One by One)**
   - This creates the sequential loop so one customer is processed, messaged, delayed, then the next customer is processed.

10. **Optional but recommended: add validation before sending**
   - Although not present in the original workflow, consider inserting an **IF** node before the HTTP request to confirm:
     - the customer has a non-empty mobile or phone
     - the AI text is non-empty
   - This avoids wasted API calls and improves observability.

11. **Configure credentials**
   - **Odoo credentials**
     - Base URL
     - Database name if required
     - Username/email
     - Password or API key depending on your Odoo setup
     - Permissions to read `account.move` and `res.partner`
   - **OpenAI credentials**
     - API key with access to the selected model
   - **Evolution API**
     - Correct endpoint URL
     - Valid API key
     - Confirm expected payload schema with your installed Evolution API version

12. **Test the workflow**
   - Run the invoice fetch node manually first.
   - Confirm returned invoices include `partner_id`.
   - Run the contact node and confirm `name`, `mobile`, or `phone` are present.
   - Run the OpenAI node and inspect the actual output path.
   - Run the HTTP node with a controlled test number before enabling the full loop.

13. **Activate the workflow**
   - Once validated, activate the workflow so the schedule starts running daily.

### Input/Output Expectations
- **Input at entry point:** none; it is schedule-driven.
- **Expected Odoo invoice fields:** at minimum `date` and `partner_id`.
- **Expected Odoo partner fields:** `name`, `mobile`, `phone`.
- **Expected OpenAI output:** a single final text message accessible in the returned response structure.
- **Expected WhatsApp API payload:** JSON containing recipient number and message text.

### Sub-workflow Setup
- This workflow does **not** use any sub-workflow or Execute Workflow node.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| This workflow automatically detects customers who made a purchase exactly 10 days ago, generates a highly personalized, natural-sounding Arabic promotional message using OpenAI, and sends it directly via WhatsApp. It includes an anti-ban loop to pace the messages. | Workflow purpose |
| Setup steps: 1. Odoo: Connect your credentials to fetch Invoices and Contacts. 2. OpenAI: Add your API Key to the AI Node to generate the custom copywriting. 3. WhatsApp: Update the HTTP Request node with your Evolution API credentials, or replace it with another WhatsApp provider node. | Operational setup |
| There is a logic discrepancy to verify: the title and notes say 10 days, but the date expression uses `$today.minus({ days: 9 })`. | Implementation note |
| The phone normalization logic is specifically tailored for Egyptian numbers by converting local prefixes to country code `20`. | Localization note |
| Send errors do not stop the workflow because the WhatsApp HTTP node is configured to continue on error. | Reliability/monitoring note |