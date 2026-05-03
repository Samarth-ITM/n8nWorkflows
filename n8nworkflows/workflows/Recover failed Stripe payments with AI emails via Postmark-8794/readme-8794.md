Recover failed Stripe payments with AI emails via Postmark

https://n8nworkflows.xyz/workflows/recover-failed-stripe-payments-with-ai-emails-via-postmark-8794


# Recover failed Stripe payments with AI emails via Postmark

### 1. Workflow Overview

This workflow automates the recovery of failed Stripe subscription payments by sending personalized, AI-generated email reminders to customers via Postmark. It targets subscription invoices where automatic payment attempts have failed, aiming to politely but urgently encourage customers to complete their payment and avoid service interruption.

The workflow consists of the following logical blocks:

- **1.1 Input Reception:** Receives Stripe webhook events related to invoices.
- **1.2 Event Filtering:** Filters to process only relevant Stripe invoice events.
- **1.3 Data Extraction & Preparation:** Parses and structures invoice data from the webhook payload.
- **1.4 Conditional Logic:** Ensures only failed, recurring subscription invoices with automatic charge collection proceed.
- **1.5 AI Email Generation:** Uses an AI agent (Langchain + OpenAI) to craft a personalized recovery email in HTML format.
- **1.6 Email JSON Parsing:** Converts the AI agent’s raw JSON string output into a structured JSON object.
- **1.7 Email Dispatch:** Sends the generated email via Postmark API.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:** Receives incoming HTTP POST requests from Stripe containing event data.
- **Nodes Involved:** `Webhook`
- **Node Details:**
  - Type: Webhook node — accepts HTTP POST requests.
  - Configuration: Path set to a unique webhook ID (`7c277748-d5fe-46f1-a33b-771acf6a7fe5`), HTTP method POST.
  - Inputs: External Stripe webhook POST requests.
  - Outputs: Raw JSON payload from Stripe.
  - Edge Cases: Missing or malformed webhook data; ensure webhook path is correctly set in Stripe dashboard.
  - Sticky Note: Reminds to replace the mock UUID path with the actual generated n8n webhook path.

#### 2.2 Event Filtering

- **Overview:** Filters incoming webhook payloads to only continue if the event is a Stripe invoice event.
- **Nodes Involved:** `Filter`
- **Node Details:**
  - Type: Filter node.
  - Configuration: Checks two conditions must be true:
    - `body.data.object.object` equals `"invoice"`
    - `body.object` equals `"event"`
  - Inputs: Output from `Webhook`.
  - Outputs: Passes data only if both conditions match.
  - Edge Cases: Ignores non-invoice events or non-Stripe webhook calls.
  - Sticky Note: Suggests adding this filter to guard against non-Stripe webhook calls.

#### 2.3 Data Extraction & Preparation

- **Overview:** Extracts and formats relevant invoice and customer data from the webhook payload for downstream processing.
- **Nodes Involved:** `Code in JavaScript`
- **Node Details:**
  - Type: Code node using JavaScript.
  - Configuration:
    - Parses nested JSON from `body.data.object`.
    - Extracts fields:
      - Customer first and last names (split on first space).
      - Currency, customer email, hosted invoice URL.
      - Invoice number and amount (converts cents to float).
      - Event type, billing reason, collection method, account name.
      - Description prioritizes top-level; falls back to first line item description.
  - Inputs: Output from `Filter`.
  - Outputs: Structured JSON with clean fields suitable for AI prompt.
  - Edge Cases:
    - Missing fields default to empty strings.
    - Name splitting assumes one space; no handling for multiple spaces.
  - Sticky Note: Details the fields extracted and mapping logic.

#### 2.4 Conditional Logic

- **Overview:** Ensures processing continues only for failed payments on recurring subscription invoices with automatic charge collection.
- **Nodes Involved:** `If`
- **Node Details:**
  - Type: If node.
  - Configuration: All three conditions must be true:
    - Event `type` equals `"invoice.payment_failed"`
    - `billing_reason` equals `"subscription_cycle"`
    - `collection` equals `"charge_automatically"`
  - Inputs: Output from `Code in JavaScript`.
  - Outputs: True branch proceeds to AI email generation.
  - Edge Cases:
    - Prevents sending emails for non-subscription invoices, manual collection, or successful payments.
  - Sticky Note: Emphasizes filtering to target recurring subscription invoices only.

#### 2.5 AI Email Generation

- **Overview:** Generates a personalized, concise, and polite email body in HTML using AI, instructing the customer to pay the failed invoice.
- **Nodes Involved:** `AI Agent`, `OpenAI Chat Model`
- **Node Details:**
  - `OpenAI Chat Model`:
    - Type: Langchain OpenAI Chat Model node.
    - Configuration: Uses model `gpt-4.1-mini`.
    - Credentials: Uses OpenAI API credentials.
    - Output: JSON object string from AI.
  - `AI Agent`:
    - Type: Langchain Agent node.
    - Configuration:
      - Receives structured invoice data as input variables.
      - Prompt instructs to write a short, friendly, urgent email including:
        - Greeting with first name fallback.
        - Subscription/plan name parsed from description or account name.
        - Invoice number and formatted amount with currency symbol.
        - Clear call-to-action button linking to hosted invoice URL.
        - Polite closing with signature.
      - Output format strictly one JSON object with keys: `to_email`, `email_subject`, `email_body` (HTML snippet).
    - Inputs: AI language model output.
    - Outputs: Raw AI-generated JSON string.
  - Inputs: True output of `If` node.
  - Outputs: Raw AI response string to be parsed.
  - Edge Cases:
    - Missing variables replaced with empty strings.
    - Missing invoice URL replaced with a button linking to `"#"`.
    - Requires valid OpenAI credentials.
  - Sticky Notes:
    - Explains prompt logic and variable usage.
    - Reminds to replace Postmark sender email and credentials on import.

#### 2.6 Email JSON Parsing

- **Overview:** Parses the AI agent’s raw JSON string response into a proper JSON object for the email sending node.
- **Nodes Involved:** `Code in JavaScript1`
- **Node Details:**
  - Type: Code node using JavaScript.
  - Configuration:
    - Extracts JSON substring from AI agent’s text output, handling possible markdown code fences.
    - Parses JSON to an object with keys: `to_email`, `email_subject`, `email_body`.
    - Throws error if JSON cannot be parsed.
  - Inputs: Output from `AI Agent`.
  - Outputs: Valid JSON object for email sending.
  - Edge Cases:
    - Failures if AI response is malformed or missing JSON.
  - Sticky Note: Notes this node extracts and parses JSON string from AI output.

#### 2.7 Email Dispatch

- **Overview:** Sends the generated email to the customer using Postmark API.
- **Nodes Involved:** `HTTP Request1`
- **Node Details:**
  - Type: HTTP Request node.
  - Configuration:
    - Method: POST.
    - URL: `https://api.postmarkapp.com/email`
    - Headers: `Accept: application/json`, `Content-Type: application/json`
    - Body parameters:
      - `From`: Uses environment variable `POSTMARK_FROM_EMAIL`.
      - `To`: `to_email` from parsed AI output.
      - `Subject`: `email_subject` from parsed AI output.
      - `HtmlBody`: `email_body` from parsed AI output.
      - `MessageStream`: `"n8n_demo"`
    - Authentication: Predefined Postmark API credentials.
  - Inputs: Valid email JSON from previous node.
  - Outputs: Postmark API response.
  - Edge Cases:
    - Requires valid Postmark API credentials.
    - Network or API errors may cause email sending failure.
  - Sticky Note: Reminds to replace sender email and attach Postmark credentials.

---

### 3. Summary Table

| Node Name          | Node Type                 | Functional Role                               | Input Node(s)      | Output Node(s)     | Sticky Note                                                                                     |
|--------------------|---------------------------|-----------------------------------------------|--------------------|--------------------|------------------------------------------------------------------------------------------------|
| Webhook            | Webhook                   | Receives Stripe webhook HTTP POST events      | External Stripe     | Filter             | Reminds to replace mock UUID path with actual webhook path                                     |
| Filter             | Filter                    | Filters only Stripe invoice events             | Webhook            | Code in JavaScript  | Suggests filter to guard against non-Stripe webhooks                                          |
| Code in JavaScript  | Code (JavaScript)         | Extracts and maps invoice & customer data      | Filter             | If                 | Details fields extracted from `body.data.object`                                              |
| If                 | If                        | Checks for failed recurring subscription payment | Code in JavaScript  | AI Agent           | Filters to only email on failed recurring subscription invoices                               |
| OpenAI Chat Model   | Langchain OpenAI Chat     | Provides AI language model for email generation | AI Agent           | AI Agent           | -                                                                                              |
| AI Agent           | Langchain Agent           | Generates AI email JSON from invoice data      | If, OpenAI Chat Model | Code in JavaScript1 | Explains prompt and variables; reminds Postmark sender replacement                            |
| Code in JavaScript1 | Code (JavaScript)         | Parses AI JSON response string to JSON object  | AI Agent           | HTTP Request1      | Extracts JSON object from AI agent output                                                    |
| HTTP Request1      | HTTP Request (Postmark)   | Sends the email via Postmark API                | Code in JavaScript1 | -                  | Reminds to replace sender email and attach Postmark credentials                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**
   - Type: Webhook
   - Path: Use unique path or UUID, e.g., `stripe-failed-payment`
   - HTTP Method: POST
   - Purpose: Receive Stripe webhook events.

2. **Create Filter Node**
   - Type: Filter
   - Condition 1: `body.data.object.object` equals `"invoice"`
   - Condition 2: `body.object` equals `"event"`
   - Connect Webhook → Filter
   - Purpose: Guard to process only Stripe invoice events.

3. **Create Code Node (Data Extraction)**
   - Type: Code (JavaScript)
   - Paste code that:
     - Extracts from `$json.body.data.object`
     - Parses customer full name into `firstName` and `lastName`
     - Extracts `currency`, `customer_email`, `hosted_invoice_url`, `invoice_number`, `invoice_amount` (divide by 100)
     - Extracts `type`, `billing_reason`, `collection`, `account_name`
     - Extracts `description` from invoice or fallback line item
   - Connect Filter → Code
   - Purpose: Prepare structured data for AI prompt.

4. **Create If Node**
   - Type: If
   - Conditions (all must be true):
     - `type` equals `invoice.payment_failed`
     - `billing_reason` equals `subscription_cycle`
     - `collection` equals `charge_automatically`
   - Connect Code → If
   - Purpose: Filter for failed automatic subscription payments.

5. **Create OpenAI Chat Model Node**
   - Type: `@n8n/n8n-nodes-langchain.lmChatOpenAi`
   - Model: `gpt-4.1-mini`
   - Credentials: Attach OpenAI API credentials
   - Purpose: Provide AI language model for generating email content.

6. **Create AI Agent Node**
   - Type: `@n8n/n8n-nodes-langchain.agent`
   - Parameters:
     - Input text with variables from extracted data (e.g., `firstName`, `invoice_number`, etc.)
     - Prompt instructing to write friendly, urgent payment-failed email in minimal HTML with button linking to invoice URL.
     - Output: Single JSON object with keys: `to_email`, `email_subject`, `email_body`.
     - System message indicating role as Billing & Payments Manager.
   - Connect If (true branch) → AI Agent
   - Connect OpenAI Chat Model → AI Agent (as language model)
   - Purpose: Generate personalized email JSON from invoice data.

7. **Create Code Node (Parse AI JSON)**
   - Type: Code (JavaScript)
   - Paste code to:
     - Extract JSON substring from AI raw text response (strip code fences)
     - Parse to JSON object with keys: `to_email`, `email_subject`, `email_body`
     - Throw error if parsing fails
   - Connect AI Agent → Code (Parse AI JSON)
   - Purpose: Convert AI output string into usable JSON.

8. **Create HTTP Request Node (Postmark)**
   - Type: HTTP Request
   - Method: POST
   - URL: `https://api.postmarkapp.com/email`
   - Headers:
     - Accept: application/json
     - Content-Type: application/json
   - Body Parameters (JSON):
     - `From`: Set to environment variable `POSTMARK_FROM_EMAIL`
     - `To`: `{{ $json.to_email }}`
     - `Subject`: `{{ $json.email_subject }}`
     - `HtmlBody`: `{{ $json.email_body }}`
     - `MessageStream`: `n8n_demo` (or your Postmark stream)
   - Authentication: Use Postmark API credentials
   - Connect Code (Parse AI JSON) → HTTP Request
   - Purpose: Send the email via Postmark.

9. **Final Connections**
   - Webhook → Filter → Code (Data Extraction) → If → AI Agent → Code (Parse AI JSON) → HTTP Request
   - OpenAI Chat Model → AI Agent (as language model)

10. **Credential Setup**
    - Configure OpenAI API credentials with your API key.
    - Configure Postmark API credentials with your API token.
    - Set environment variable `POSTMARK_FROM_EMAIL` to your verified sender email.

---

### 5. General Notes & Resources

| Note Content                                                                                                    | Context or Link                                                                                          |
|----------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| The workflow extracts and formats Stripe invoice data, splitting customer names on the first space only.       | See Code in JavaScript node sticky note for field mapping details.                                      |
| The AI-generated email uses minimal HTML snippet with a styled button containing the hosted invoice URL.       | Prompt details in AI Agent node; important for email rendering consistency.                              |
| Ensure your Postmark sender email and API credentials are correctly configured to avoid sending failures.      | Sticky notes on HTTP Request node highlight this requirement.                                           |
| The webhook path is a mock UUID and must be replaced with the actual n8n webhook URL configured in Stripe.     | Sticky note on Webhook node emphasizes path replacement.                                                |
| AI response parsing expects a single JSON object within the response; malformed or missing JSON causes errors. | Code in JavaScript1 node handles this; monitor for parsing exceptions in logs.                           |
| Project credits: This workflow leverages n8n’s Langchain integration with OpenAI GPT-4 for AI content creation. |                                                                                                         |

---

**Disclaimer:** The provided text is exclusively derived from an automated workflow created using n8n, a tool for integration and automation. This processing strictly complies with current content policies and does not include any illegal, offensive, or protected elements. All data handled is legal and publicly accessible.