Generate Daily E-Commerce Order Reports with Supabase, GPT-4.1 and Gmail

https://n8nworkflows.xyz/workflows/generate-daily-e-commerce-order-reports-with-supabase--gpt-4-1-and-gmail-4406


# Generate Daily E-Commerce Order Reports with Supabase, GPT-4.1 and Gmail

### 1. Workflow Overview

This workflow automates the generation and email delivery of a daily summary report for recent e-commerce orders stored in a Supabase database. It targets e-commerce administrators or business intelligence teams who need a concise, automatically generated overview and detailed listing of new orders placed in the last 24 hours.

The workflow is logically divided into these main blocks:

- **1.1 Scheduled Trigger and Input Setup:** Starts the workflow daily at 8 AM, sets the sender email address.
- **1.2 Data Retrieval from Supabase:** Fetches comprehensive data from four Supabase tables — orders, order items, clients (users), and products.
- **1.3 AI Processing and Report Generation:** Uses an AI agent (LangChain with GPT-4.1) to process the data, filter recent orders, generate a plain text summary and detailed order list, formatted for email.
- **1.4 Email Delivery:** Sends the generated report as a plain text email via Gmail.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Trigger and Input Setup

- **Overview:**  
  This block triggers the workflow daily at a specified time and sets the recipient email address for the outgoing report.

- **Nodes Involved:**  
  - Daily 8am  
  - Set Sender Email

- **Node Details:**  

  - **Daily 8am**  
    - Type: Schedule Trigger  
    - Role: Initiates execution every day at 8:00 AM Europe/Madrid time.  
    - Configuration: Set to trigger exactly at hour 8 (8 AM).  
    - Inputs: None (trigger node).  
    - Outputs: Passes trigger event to "Set Sender Email".  
    - Edge cases: If n8n instance is down or paused at trigger time, execution will be delayed or missed.

  - **Set Sender Email**  
    - Type: Set  
    - Role: Defines the recipient email address variable (`toEmail`) for downstream use.  
    - Configuration: Assigns a static string email value (placeholder "setEmailHere" which must be replaced with the actual recipient address).  
    - Inputs: From "Daily 8am".  
    - Outputs: Passes the data including `toEmail` to "AI Agent".  
    - Edge cases: If the email address is not set or invalid, the email sending step will fail.

---

#### 2.2 Data Retrieval from Supabase

- **Overview:**  
  This block fetches all necessary raw data from Supabase tables to be processed for the report: orders, order items, clients (users), and products.

- **Nodes Involved:**  
  - Get Orders  
  - Get Order Items  
  - Get Clients  
  - Get Products

- **Node Details:**

  - **Get Orders**  
    - Type: Supabase Tool  
    - Role: Retrieves all records from the `orders` table.  
    - Configuration: Operation set to "getAll" with an optional limit parameter (default undefined, can be overridden).  
    - Credentials: Connected to Supabase account "PastaDemo".  
    - Inputs: Triggered by the AI Agent node's tool call.  
    - Outputs: Provides the full orders dataset for AI processing.  
    - Edge cases: Supabase connection/auth failures, timeout, empty table.

  - **Get Order Items**  
    - Type: Supabase Tool  
    - Role: Retrieves all records from the `order_items` table.  
    - Configuration: Operation "getAll".  
    - Credentials: Same Supabase account.  
    - Inputs: AI Agent tool call.  
    - Outputs: Order line items dataset.  
    - Edge cases: Similar to Get Orders.

  - **Get Clients**  
    - Type: Supabase Tool  
    - Role: Retrieves all records from the `users` table (clients).  
    - Configuration: Operation "getAll".  
    - Credentials: Same Supabase account.  
    - Inputs: AI Agent tool call.  
    - Outputs: Client user data.  
    - Edge cases: Same as others.

  - **Get Products**  
    - Type: Supabase Tool  
    - Role: Retrieves all records from the `products` table.  
    - Configuration: Operation "getAll".  
    - Credentials: Same Supabase account.  
    - Inputs: AI Agent tool call.  
    - Outputs: Product details dataset.  
    - Edge cases: Same as others.

---

#### 2.3 AI Processing and Report Generation

- **Overview:**  
  This block uses a LangChain AI agent with GPT-4.1 to aggregate, filter, and format the data into a plain text summary report and detailed order list suitable for email delivery.

- **Nodes Involved:**  
  - AI Agent  
  - OpenAI Chat Model

- **Node Details:**

  - **OpenAI Chat Model**  
    - Type: LangChain OpenAI Chat model  
    - Role: Provides GPT-4.1 large language model capabilities to the AI Agent.  
    - Configuration: Model explicitly set to "gpt-4.1".  
    - Credentials: Uses OpenAI API credentials.  
    - Inputs: Connected as language model backend to AI Agent.  
    - Outputs: Provides AI-generated text responses.  
    - Edge cases: API key rate limits, quota exhaustion, network issues.

  - **AI Agent**  
    - Type: LangChain Agent  
    - Role: Orchestrates calls to Supabase data retrieval tools and OpenAI model to generate the report.  
    - Configuration:  
      - Prompt defines the agent as an intelligent assistant tasked with:  
        - Retrieving data from Supabase tables via the specific tools (Get Orders, Get Order Items, Get Clients, Get Products).  
        - Filtering orders placed in the last 24 hours relative to the current time.  
        - Generating a plain text summary and detailed list in a strict format (no markdown, bullets, or special formatting, only newlines and spaces).  
        - Sending the email using the "Send Gmail Summary" tool with subject "Daily Order Summary - Last 24 Hours".  
      - The prompt includes an example email body in Italian localized to Rome timezone.  
    - Inputs: Receives `toEmail` from "Set Sender Email".  
    - Outputs: Calls four Supabase tools and one Gmail tool as sub-tools; outputs data to "Send Gmail Summary".  
    - Edge cases:  
      - Failure in any tool calls (Supabase or Gmail) will break the flow.  
      - Expression or prompt syntax errors.  
      - Insufficient or malformed input data (e.g., no recent orders).  
      - API limits or timeouts in OpenAI or Supabase.

---

#### 2.4 Email Delivery

- **Overview:**  
  Sends the AI-generated plain text order summary email to the configured recipient via Gmail.

- **Nodes Involved:**  
  - Send Gmail Summary

- **Node Details:**

  - **Send Gmail Summary**  
    - Type: Gmail Tool  
    - Role: Sends the final email containing the daily order report.  
    - Configuration:  
      - Recipient email dynamically set from `$json.toEmail`.  
      - Email subject and message body are dynamically populated by the AI Agent tool outputs.  
      - Email type set to plain text.  
      - Gmail OAuth2 credentials used for authentication.  
    - Inputs: Called as a tool by the AI Agent.  
    - Outputs: Email sent confirmation or error.  
    - Edge cases:  
      - Authentication failure with Gmail OAuth2.  
      - Invalid recipient email address.  
      - Network or API errors.  
      - Email delivery delays or spam filtering.

---

### 3. Summary Table

| Node Name         | Node Type                        | Functional Role                              | Input Node(s)        | Output Node(s)      | Sticky Note                                                                                 |
|-------------------|---------------------------------|----------------------------------------------|----------------------|---------------------|---------------------------------------------------------------------------------------------|
| Daily 8am         | Schedule Trigger                | Triggers workflow daily at 8 AM               | -                    | Set Sender Email     |                                                                                             |
| Set Sender Email  | Set                            | Sets recipient email address (`toEmail`)     | Daily 8am            | AI Agent            | - Connect to your Supabase account to pull data from tables<br>- Update the sender email<br>- Customize the AI agent prompt to fit the use case |
| Get Orders        | Supabase Tool                  | Retrieves orders data from Supabase           | AI Agent (tool call) | AI Agent            |                                                                                             |
| Get Order Items   | Supabase Tool                  | Retrieves order items data                      | AI Agent (tool call) | AI Agent            |                                                                                             |
| Get Clients       | Supabase Tool                  | Retrieves clients (users) data                  | AI Agent (tool call) | AI Agent            |                                                                                             |
| Get Products      | Supabase Tool                  | Retrieves products data                         | AI Agent (tool call) | AI Agent            |                                                                                             |
| OpenAI Chat Model | LangChain OpenAI Chat Model    | Provides GPT-4.1 language model                 | AI Agent (language model) | AI Agent        |                                                                                             |
| AI Agent          | LangChain Agent                | Coordinates data retrieval, AI processing, and email sending | Set Sender Email, Supabase tools, OpenAI Chat Model | Send Gmail Summary |                                                                                             |
| Send Gmail Summary| Gmail Tool                    | Sends daily order summary email                 | AI Agent (tool call) | -                   |                                                                                             |
| Sticky Note       | Sticky Note                   | Workflow notes and reminders                    | -                    | -                   | - Connect to your Supabase account to pull data from tables<br>- Update the sender email<br>- Customize the AI agent prompt to fit the use case |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a Schedule Trigger node:**  
   - Name: "Daily 8am"  
   - Set to trigger once daily at 8:00 AM (Europe/Madrid timezone).  
   - Save.

3. **Add a Set node:**  
   - Name: "Set Sender Email"  
   - Add a string field assignment named `toEmail`.  
   - Set the value to the recipient email address (e.g., your email).  
   - Connect the output of "Daily 8am" to this node.

4. **Add Supabase Tool nodes for data retrieval:**  
   For each table (`orders`, `order_items`, `users`, `products`), create a Supabase Tool node:

   - Name accordingly: "Get Orders", "Get Order Items", "Get Clients", "Get Products".  
   - Operation: `getAll`  
   - Table name: respective table.  
   - Credentials: Configure with your Supabase API credentials.  
   - No limit set (or set as needed).  
   - These nodes will not connect in sequence but will be called as tools by the AI agent.

5. **Add an OpenAI Chat Model node:**  
   - Name: "OpenAI Chat Model"  
   - Model: Select or enter "gpt-4.1".  
   - Credentials: Use OpenAI API credentials.

6. **Add an AI Agent node:**  
   - Name: "AI Agent"  
   - Set type to LangChain Agent.  
   - Configure the prompt as follows (adapted from the original):  
     - Define the agent as an assistant to generate daily order summaries.  
     - Specify that it must call the Supabase tools "Get Orders", "Get Order Items", "Get Clients", "Get Products" to retrieve data.  
     - Instruct it to filter orders placed within the last 24 hours based on current date/time.  
     - Generate a plain text summary with total orders and revenue, plus detailed order info (order ID, date/time, client name, total value, list of products with quantity and price).  
     - Formatting instructions: plain text only, use newlines, no markdown or special chars, indent lists with spaces.  
     - Specify it must call the tool "Send Gmail Summary" to send the email.  
     - Set email subject: "Daily Order Summary - Last 24 Hours".  
     - Pass the `toEmail` from the "Set Sender Email" node as input.  
   - Connect the output of "Set Sender Email" to the AI Agent node’s main input.  
   - Configure the AI Agent node to use the OpenAI Chat Model node as its language model.  
   - Add the Supabase Tool nodes and Gmail node as tools accessible to the AI Agent.

7. **Add a Gmail Tool node:**  
   - Name: "Send Gmail Summary"  
   - Email type: plain text.  
   - Recipient email: expression referencing `toEmail` (e.g., `{{$json["toEmail"]}}`).  
   - Subject and message body will be dynamically set by the AI Agent tool calls.  
   - Credentials: Configure Gmail OAuth2 credentials.  
   - This node will be called as a tool by the AI Agent.

8. **Connections overview:**  
   - "Daily 8am" → "Set Sender Email" → "AI Agent"  
   - "AI Agent" calls:  
     - "Get Orders" (tool)  
     - "Get Order Items" (tool)  
     - "Get Clients" (tool)  
     - "Get Products" (tool)  
     - "Send Gmail Summary" (tool)  
   - "OpenAI Chat Model" connected as language model backend to "AI Agent".

9. **Set workflow timezone to Europe/Madrid.**

10. **Save and activate the workflow.**

---

### 5. General Notes & Resources

| Note Content                                                                                                 | Context or Link                                           |
|--------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------|
| Connect to your Supabase account to pull data from your specific tables (`orders`, `order_items`, etc.).      | Workflow setup instruction                                |
| Update the sender email address in the "Set Sender Email" node to the actual email recipient.                | Email delivery setup                                      |
| Customize the AI Agent prompt to fit your specific reporting needs or localization (e.g., language, timezone).| AI prompt customization                                   |
| Example email formatting is in Italian localized to Rome timezone (CEST) but can be adapted as needed.       | Localization and formatting example                       |
| Use Gmail OAuth2 credentials for secure email sending; ensure proper OAuth consent and scopes are set.       | Gmail API authentication                                  |
| The AI Agent node uses LangChain integration with GPT-4.1 model for advanced reasoning and tool orchestration.| n8n LangChain AI integration documentation               |

---

**Disclaimer:**  
The provided text is exclusively derived from an automated n8n workflow and complies fully with content policies. It contains no illegal, offensive, or protected content. All manipulated data is legal and public.