Send personalized SaaS welcome emails with Stripe, Pinecone, GPT-4o, and Gmail

https://n8nworkflows.xyz/workflows/send-personalized-saas-welcome-emails-with-stripe--pinecone--gpt-4o--and-gmail-13861


# Send personalized SaaS welcome emails with Stripe, Pinecone, GPT-4o, and Gmail

### 1. Workflow Overview

This workflow automates the end-to-end onboarding process for a SaaS platform. It is triggered by a successful customer payment via Stripe and performs a series of intelligent operations: converting payment data, retrieving specific subscription tier benefits from a vector database (Pinecone) using AI, generating a hyper-personalized HTML welcome email with GPT-4o, and finally logging the transaction in Google Sheets.

**Logical Blocks:**
*   **1.1 Stripe Payment Trigger:** Captures the `checkout.session.completed` event.
*   **1.2 Data Normalization:** Converts currency units (pence to dollars) and extracts relevant customer metadata.
*   **1.3 AI Intelligence & Retrieval:** Uses an AI Agent with RAG (Retrieval-Augmented Generation) to fetch plan details from Pinecone and draft the email.
*   **1.4 Communication & Archiving:** Sends the generated content via Gmail and records the new subscription in a master Google Sheet.

---

### 2. Block-by-Block Analysis

#### 2.1 Stripe Payment Trigger
*   **Overview:** The entry point of the workflow, monitoring for successful transactions.
*   **Nodes Involved:** `Stripe Trigger`.
*   **Node Details:**
    *   **Type:** Stripe Trigger
    *   **Configuration:** Set to listen for the `checkout.session.completed` event.
    *   **Output:** Returns a JSON object containing customer details, payment totals, and session information.
    *   **Edge Cases:** Unsuccessful payments or incomplete checkouts will not trigger the workflow.

#### 2.2 Extract Customer Details
*   **Overview:** Cleans the raw Stripe data and prepares variables for the AI Agent.
*   **Nodes Involved:** `Amount conversion`, `Get user_details`.
*   **Node Details:**
    *   **Amount conversion (Code Node):** 
        *   *Function:* Divides `amount_total` by 100 to convert from cents/pence to a standard decimal currency format.
        *   *Expression:* `const amount = amountInPence / 100;`
    *   **Get user_details (Set Node):** 
        *   *Function:* Maps specific Stripe fields (Name, Email) and the converted Amount into a structured object.
        *   *Variables:* `{{ $json.amount }}`, `{{ $('Stripe Trigger').item.json.data.object.customer_details.name }}`, etc.

#### 2.3 AI Email Generation
*   **Overview:** The "brain" of the workflow. It uses a LangChain-powered agent to look up what the customer actually bought and write a custom email.
*   **Nodes Involved:** `AI Agent`, `GPT-4o`, `Pinecone Tool`, `OpenAI Embeddings`.
*   **Node Details:**
    *   **AI Agent:** Configured as a "Onboarding Assistant." The system prompt instructs it to calculate a renewal date (today + 30 days), use Pinecone to find plan features, and output a specific JSON schema.
    *   **GPT-4o (Chat Model):** Provides the reasoning and language capabilities. Set to `json_object` format.
    *   **Pinecone Tool:** A retrieval tool that allows the agent to search the "leadpulse" database for plan-specific details based on the payment amount.
    *   **OpenAI Embeddings:** Necessary for the Pinecone tool to perform vector searches on the incoming query.
    *   **Potential Failures:** Pinecone namespace mismatch, OpenAI API quota limits, or AI failing to return valid JSON.

#### 2.4 Send & Log
*   **Overview:** Executes the final actions: parsing the AI output, notifying the user, and updating internal records.
*   **Nodes Involved:** `Parse AI Output`, `Send Welcome Email`, `Log customer details`.
*   **Node Details:**
    *   **Parse AI Output (Code Node):** 
        *   *Function:* Uses `JSON.parse()` to turn the AI's string response into usable n8n fields (`tier`, `renewal_date`, `subject`, `body`).
    *   **Send Welcome Email (Gmail):** 
        *   *Function:* Sends a personalized email using the AI-generated subject and body.
        *   *Configuration:* HTML body is supported via the `message` field.
    *   **Log customer details (Google Sheets):** 
        *   *Function:* Appends a new row to a spreadsheet.
        *   *Data logged:* Name, Email, Plan Tier, Status (Active), Renewal Date, and Subscription Date.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Stripe Trigger** | Stripe Trigger | Workflow Entry | (None) | Amount conversion | **1. Stripe Payment Trigger** |
| **Amount conversion** | Code | Data Formatting | Stripe Trigger | Get user_details | **2. Extract Customer Details** |
| **Get user_details** | Set | Variable Mapping | Amount conversion | AI Agent | **2. Extract Customer Details** |
| **AI Agent** | AI Agent | Strategy & Drafting | Get user_details | Parse AI Output | **3. AI Email Generation** |
| **GPT-4o** | Chat OpenAI | AI Reasoning | (AI Agent) | (AI Agent) | **3. AI Email Generation** |
| **Pinecone Tool** | Pinecone Tool | Data Retrieval | (AI Agent) | (AI Agent) | **3. AI Email Generation** |
| **OpenAI Embeddings** | Embeddings | Vector Processing | (Pinecone Tool) | (Pinecone Tool) | **3. AI Email Generation** |
| **Parse AI Output** | Code | String-to-JSON | AI Agent | Send Welcome Email | **4. Send & Log** |
| **Send Welcome Email** | Gmail | Communication | Parse AI Output | Log customer details | **4. Send & Log** |
| **Log customer details**| Google Sheets | Database Logging | Send Welcome Email | (None) | **4. Send & Log** |

---

### 4. Reproducing the Workflow from Scratch

1.  **Preparation:** Obtain API keys for Stripe, OpenAI, Pinecone, Google Sheets (OAuth2), and Gmail (OAuth2). Ensure your Pinecone index is populated with documents describing your SaaS tiers.
2.  **Trigger Setup:** Create a **Stripe Trigger** node. Set the event to `checkout.session.completed`.
3.  **Data Processing:** 
    *   Add a **Code Node** named "Amount conversion". Use JavaScript to divide `$input.first().json.data.object.amount_total` by 100.
    *   Add a **Set Node** named "Get user_details". Create three string assignments: `amount` (from previous node), `name`, and `email` (extracted from the Stripe Trigger output).
4.  **AI Infrastructure:** 
    *   Add an **AI Agent** node. In the System Message, define the persona, the JSON output format required (`tier`, `renewal_date`, `subject`, `body`), and the logic for the 30-day renewal calculation.
    *   Connect a **Chat OpenAI** node (GPT-4o) to the "Model" input of the Agent.
    *   Connect a **Pinecone Tool** to the "Tools" input of the Agent. Provide your Index and Namespace.
    *   Connect an **OpenAI Embeddings** node to the Pinecone Tool.
5.  **Output Parsing:** 
    *   The Agent returns a string. Add a **Code Node** named "Parse AI Output". Use `JSON.parse($input.first().json.output)` to map the fields into n8n JSON format.
6.  **Final Actions:** 
    *   Add a **Gmail Node**. Set the operation to "Send". Map the "Send To" to the email from Step 3, and the "Subject"/"Body" to the outputs from Step 5.
    *   Add a **Google Sheets Node**. Set the operation to "Append". Map columns for Name, Email, Tier, Renewal Date, and Subscription Date (use `{{ $now }}`).
7.  **Connection:** Link the nodes in the order described in the Summary Table.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| **Requirements** | Stripe account, Pinecone account with tier knowledge base, OpenAI account, Gmail account, Google Sheets |
| **Workflow Logic** | This template automates the entire onboarding experience for a SaaS product with multiple subscription tiers. |
| **Setup Detail** | Ensure your Pinecone index contains "tier chunks" so the AI can distinguish between (e.g.) Basic and Pro plans. |
| **Google Sheets Headers** | Required columns: Name, Email, Tier, Subscription Date, Renewal Date, Status, Renewal Status. |