Telegram Payment, Invoicing and Refund Workflow for Stars

https://n8nworkflows.xyz/workflows/telegram-payment--invoicing-and-refund-workflow-for-stars-2461


# Telegram Payment, Invoicing and Refund Workflow for Stars

### 1. Workflow Overview

This workflow automates the entire process of handling **Telegram Stars payments, invoicing, and refunds** using n8n. It targets businesses, developers, and service providers who want to integrate Telegram Stars as a payment method within Telegram bots. The workflow manages invoice creation and sending, pre-checkout payment approvals, transaction recording, and refunds, all orchestrated through Telegram and Google Sheets.

The workflow is logically divided into the following blocks:

- **1.1 Telegram Event Reception:** Receives incoming Telegram messages and triggers the workflow.
- **1.2 Payment Initiation and Invoice Sending:** Prepares invoice data and sends invoices to users via Telegram.
- **1.3 Payment Processing and Approval:** Handles pre-checkout updates and approves payments based on Telegram events.
- **1.4 Transaction Logging:** Records payment charge IDs and related details in Google Sheets for tracking.
- **1.5 Refund Handling:** Manages refund requests via manual triggers and processes refunds through Telegram API.
- **1.6 Control Flow and Logic Routing:** Uses switches and logic nodes to route different types of Telegram events and payment states.

---

### 2. Block-by-Block Analysis

#### 1.1 Telegram Event Reception

- **Overview:**  
  This block initiates the workflow by listening for Telegram updates, extracting necessary data, and preparing the context for payment processing.

- **Nodes Involved:**  
  - Telegram Trigger (disabled)  
  - Data for Invoice  
  - Start Payment Workflow  
  - Execute Workflow Trigger  
  - Trigger Data  
  - Chat ID  
  - Bot API token

- **Node Details:**

  - **Telegram Trigger**  
    - Type: Telegram Trigger  
    - Role: Listens to incoming Telegram updates (messages, commands).  
    - Configuration: Uses webhook with an assigned webhook ID. Disabled by default (likely for controlled activation).  
    - Input: External Telegram updates  
    - Output: Raw Telegram data forwarded downstream.  
    - Edge cases: Webhook misconfiguration, Telegram API downtime.

  - **Data for Invoice**  
    - Type: Set  
    - Role: Prepares and formats data required for invoice creation, including chat ID and action names.  
    - Configuration: Sets variables such as chat ID extracted from Telegram update; includes action identifiers to guide next steps.  
    - Input: Telegram Trigger output  
    - Output: Structured invoice data for payment workflow initiation.  
    - Edge cases: Missing or malformed Telegram data (e.g., no chat ID).

  - **Start Payment Workflow**  
    - Type: Execute Workflow  
    - Role: Calls a sub-workflow dedicated to handling payment-related logic.  
    - Configuration: No parameters shown, assumed to accept invoice data as input.  
    - Input: Data for Invoice node output  
    - Output: Payment workflow execution results.  
    - Edge cases: Sub-workflow failure, parameter mismatches.

  - **Execute Workflow Trigger**  
    - Type: Execute Workflow Trigger  
    - Role: Executes another workflow triggered by external events; likely used for modular flow control.  
    - Input: None specified  
    - Output: Trigger data for downstream processing.

  - **Trigger Data**  
    - Type: Set  
    - Role: Sets or formats data received from Execute Workflow Trigger for further processing.  
    - Input: Execute Workflow Trigger output  
    - Output: Structured data for Chat ID node.

  - **Chat ID**  
    - Type: Set  
    - Role: Extracts and sets the Telegram chat ID necessary for sending messages or invoices.  
    - Input: Trigger Data output  
    - Output: Chat ID variable for authentication and requests.

  - **Bot API token**  
    - Type: Set  
    - Role: Sets Telegram Bot API token for authentication with Telegram API.  
    - Input: Chat ID output  
    - Output: Credentials bundled with data for Telegram API calls.

---

#### 1.2 Payment Initiation and Invoice Sending

- **Overview:**  
  Prepares invoice details and sends invoices to users via Telegram's payment API.

- **Nodes Involved:**  
  - Invoice Data  
  - Send Invoice  
  - Add your logic  
  - Success Message  

- **Node Details:**

  - **Invoice Data**  
    - Type: Set  
    - Role: Defines invoice parameters, including product name, description, and payment amount.  
    - Input: Output from Add your logic node  
    - Output: Structured invoice data for API request.  
    - Edge cases: Incorrect pricing format, missing required invoice fields.

  - **Send Invoice**  
    - Type: HTTP Request  
    - Role: Sends the invoice via Telegram API to the user for payment.  
    - Configuration: Uses Telegram Bot API endpoint for sending invoices; configured with Bot API token credentials.  
    - Input: Invoice Data output  
    - Output: API response with invoice status.  
    - Edge cases: API request failures, authentication errors, invalid invoice data.

  - **Add your logic**  
    - Type: NoOp (No Operation)  
    - Role: Placeholder for custom user logic after invoice sending, e.g., additional validations or data manipulations.  
    - Input: Write Telegram Payment Charge ID output (see next block)  
    - Output: Passes data downstream without changes.

  - **Success Message**  
    - Type: Telegram  
    - Role: Sends a confirmation message to the user indicating successful invoice creation or payment.  
    - Input: Add your logic here output  
    - Output: Telegram message sent to user.  
    - Edge cases: Failures in sending message due to network or API issues.

---

#### 1.3 Payment Processing and Approval

- **Overview:**  
  Handles pre-checkout query updates from Telegram to approve or reject payments and captures charge IDs for record-keeping.

- **Nodes Involved:**  
  - Event swticher (Switch)  
  - Approove / Pre-Checkout Update (HTTP Request)  
  - Write Telegram Payment Charge ID (Google Sheets)  

- **Node Details:**

  - **Event swticher**  
    - Type: Switch  
    - Role: Routes incoming Telegram events based on type, distinguishing between pre-checkout queries and successful payments.  
    - Input: Actions output (bot API token context)  
    - Output: Two main branches: one to Approve Pre-Checkout, another to Write Payment Charge ID.  
    - Edge cases: Missing or unexpected event types.

  - **Approove / Pre-Checkout Update**  
    - Type: HTTP Request  
    - Role: Sends approval for pre-checkout queries to Telegram, allowing payment to proceed.  
    - Configuration: Calls Telegram API endpoint for pre-checkout approval using Bot API token.  
    - Input: Event swticher output branch  
    - Output: Telegram API response for pre-checkout approval.  
    - Edge cases: API call failures, invalid pre-checkout data.

  - **Write Telegram Payment Charge ID**  
    - Type: Google Sheets  
    - Role: Logs successful payment details, including payment charge ID and user information, into a Google Sheet for record keeping.  
    - Input: Event swticher output branch for successful payments  
    - Output: Confirmation of data written to Google Sheets.  
    - Edge cases: Google API authentication errors, rate limits, invalid sheet ID or range.

---

#### 1.4 Transaction Logging

- **Overview:**  
  Records all payment transaction details into Google Sheets to maintain transparent and accessible financial records.

- **Nodes Involved:**  
  - Write Telegram Payment Charge ID (also part of 1.3)  

- **Node Details:**  
  - See above (Write Telegram Payment Charge ID under 1.3).

---

#### 1.5 Refund Handling

- **Overview:**  
  Enables manual triggering of refund requests and automates the refund process by calling Telegram API with relevant payment and user data.

- **Nodes Involved:**  
  - Make a Refund (Manual Trigger)  
  - Bot API token (for refund)  
  - Refund Data  
  - Refund (HTTP Request)  
  - Add your Refund logic here  

- **Node Details:**

  - **Make a Refund**  
    - Type: Manual Trigger  
    - Role: Starts the refund process manually, allowing operators to initiate refunds on demand.  
    - Input: None (manual activation)  
    - Output: Triggered flow to set up refund data.

  - **Bot API token (for refund)**  
    - Type: Set  
    - Role: Sets the Telegram Bot API token specifically for refund API calls.  
    - Input: Make a Refund output  
    - Output: Credentials for refund requests.

  - **Refund Data**  
    - Type: Set  
    - Role: Prepares refund parameters including user IDs and payment charge IDs necessary to process refunds through Telegram.  
    - Input: Bot API token (for refund) output  
    - Output: Structured refund request data.

  - **Refund**  
    - Type: HTTP Request  
    - Role: Sends refund requests to Telegram payment API to refund stars to users.  
    - Configuration: Uses Telegram Bot API refund endpoint with credentials and refund data.  
    - Input: Refund Data output  
    - Output: Telegram API refund response.  
    - Edge cases: API errors, invalid payment charge IDs, network issues.

  - **Add your Refund logic here**  
    - Type: NoOp  
    - Role: Placeholder for additional refund handling logic or notifications post-refund.  
    - Input: Refund output  
    - Output: Passes data downstream or triggers other processes.

---

#### 1.6 Control Flow and Logic Routing

- **Overview:**  
  Manages routing of different Telegram actions and payment-related events through switches and logic nodes for modular, maintainable flow control.

- **Nodes Involved:**  
  - Actions (Switch)  
  - Add your logic  
  - Add your logic here  
  - Add your Refund logic here  

- **Node Details:**

  - **Actions**  
    - Type: Switch  
    - Role: Determines the type of Telegram event or user action (e.g., payment, refund, other commands) and routes the flow accordingly.  
    - Input: Bot API token output  
    - Output: Multiple branches leading to appropriate logic nodes.  
    - Edge cases: Unrecognized actions causing no routing.

  - **Add your logic**  
    - Type: NoOp  
    - Role: Custom placeholder for extending payment-related processing logic.  
    - Input: Actions output branch  
    - Output: Invoice Data node.

  - **Add your logic here**  
    - Type: NoOp  
    - Role: Custom placeholder for logic after recording payment charge ID.  
    - Input: Write Telegram Payment Charge ID output  
    - Output: Success Message node.

  - **Add your Refund logic here**  
    - Type: NoOp  
    - Role: Placeholder for additional refund processing or notifications.  
    - Input: Actions output branch for refund  
    - Output: Further refund workflow steps.

---

### 3. Summary Table

| Node Name                | Node Type               | Functional Role                              | Input Node(s)                       | Output Node(s)                       | Sticky Note                                                                                                    |
|--------------------------|-------------------------|----------------------------------------------|-----------------------------------|------------------------------------|----------------------------------------------------------------------------------------------------------------|
| Telegram Trigger          | Telegram Trigger        | Receives Telegram events                      | External webhook                  | Data for Invoice                   | Disabled by default                                                                                             |
| Data for Invoice         | Set                      | Prepares invoice data                          | Telegram Trigger                  | Start Payment Workflow             | Chat ID required. Send action name to handle it inside Payment workflow                                        |
| Start Payment Workflow    | Execute Workflow         | Executes payment sub-workflow                  | Data for Invoice                  | None specified                    |                                                                                                                |
| Execute Workflow Trigger  | Execute Workflow Trigger | Executes additional workflow trigger           | None                            | Trigger Data                      |                                                                                                                |
| Trigger Data             | Set                      | Sets data from workflow trigger                | Execute Workflow Trigger          | Chat ID                          |                                                                                                                |
| Chat ID                  | Set                      | Extracts chat ID for Telegram API calls        | Trigger Data                     | Bot API token                    |                                                                                                                |
| Bot API token            | Set                      | Sets Telegram Bot API token                     | Chat ID                         | Actions                         |                                                                                                                |
| Actions                  | Switch                   | Routes Telegram events to appropriate logic   | Bot API token                   | Add your logic, Event swticher, Add your Refund logic here |                                                                                                                |
| Add your logic           | NoOp                     | Custom placeholder for payment logic           | Actions                        | Invoice Data                    |                                                                                                                |
| Invoice Data             | Set                      | Prepares invoice parameters                     | Add your logic                  | Send Invoice                   |                                                                                                                |
| Send Invoice             | HTTP Request             | Sends invoice via Telegram API                  | Invoice Data                   | None specified                 |                                                                                                                |
| Event swticher           | Switch                   | Routes payment events (pre-checkout, payment) | Actions                       | Approove / Pre-Checkout Update, Write Telegram Payment Charge ID |                                                                                                                |
| Approove / Pre-Checkout Update | HTTP Request       | Approves pre-checkout queries                   | Event swticher                | None specified                 |                                                                                                                |
| Write Telegram Payment Charge ID | Google Sheets      | Logs payment charge ID and transaction details | Event swticher                | Add your logic here             |                                                                                                                |
| Add your logic here      | NoOp                     | Custom placeholder after logging payment       | Write Telegram Payment Charge ID | Success Message                |                                                                                                                |
| Success Message          | Telegram                 | Sends payment success confirmation message     | Add your logic here             | None specified                 |                                                                                                                |
| Make a Refund            | Manual Trigger           | Initiates refund process manually               | None                          | Bot API token (for refund)      |                                                                                                                |
| Bot API token (for refund) | Set                    | Sets API token for refund API calls              | Make a Refund                  | Refund Data                    |                                                                                                                |
| Refund Data              | Set                      | Prepares refund request data                     | Bot API token (for refund)     | Refund                        |                                                                                                                |
| Refund                   | HTTP Request             | Sends refund request via Telegram API            | Refund Data                   | None specified                 |                                                                                                                |
| Add your Refund logic here | NoOp                    | Placeholder for refund-related custom logic      | Actions                       | None specified                 |                                                                                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger Node**  
   - Type: Telegram Trigger  
   - Configure webhook with Telegram bot webhook ID  
   - Initially disable until setup is complete  
   - No additional parameters needed

2. **Add Set Node "Data for Invoice"**  
   - Extract `chat.id` from Telegram update data  
   - Add a parameter for action name to identify invoice creation  
   - Connect output of Telegram Trigger to this node

3. **Add Execute Workflow Node "Start Payment Workflow"**  
   - Set to execute the payment handling sub-workflow  
   - Pass output of "Data for Invoice" as input  
   - Connect "Data for Invoice" output here

4. **Add Execute Workflow Trigger Node "Execute Workflow Trigger"**  
   - Used for modular event triggering  
   - Connect as needed based on your integration design

5. **Add Set Node "Trigger Data"**  
   - Prepare data from executed workflow trigger for next steps  
   - Connect output of Execute Workflow Trigger here

6. **Add Set Node "Chat ID"**  
   - Extract and set chat ID for Telegram API calls  
   - Connect output of "Trigger Data" here

7. **Add Set Node "Bot API token"**  
   - Enter your Telegram Bot API token (from BotFather)  
   - Connect output of "Chat ID" here

8. **Add Switch Node "Actions"**  
   - Configure to distinguish action types (payment, refund, others) based on incoming data  
   - Connect output of "Bot API token" here

9. **Add NoOp Node "Add your logic"**  
   - Placeholder for payment-related logic customization  
   - Connect first output of "Actions" here

10. **Add Set Node "Invoice Data"**  
    - Define invoice parameters: product name, description, price, currency, payload, provider token, start parameter, photo URL (optional)  
    - Connect output of "Add your logic" here

11. **Add HTTP Request Node "Send Invoice"**  
    - Configure to send invoice via Telegram API's `sendInvoice` method  
    - Use Bot API token for authorization  
    - Set request body from "Invoice Data"  
    - Connect output of "Invoice Data" here

12. **Add Switch Node "Event swticher"**  
    - Configure to route between pre-checkout query and successful payment events  
    - Connect output of "Actions" second output here

13. **Add HTTP Request Node "Approove / Pre-Checkout Update"**  
    - Configure to call Telegram API method `answerPreCheckoutQuery` with approval  
    - Connect first output of "Event swticher" here

14. **Add Google Sheets Node "Write Telegram Payment Charge ID"**  
    - Connect to your Google Sheets account  
    - Configure to append payment charge ID, user ID, chat ID, and timestamp to a defined sheet and range  
    - Connect second output of "Event swticher" here

15. **Add NoOp Node "Add your logic here"**  
    - For post-payment logic such as notifications or further processing  
    - Connect output of "Write Telegram Payment Charge ID" here

16. **Add Telegram Node "Success Message"**  
    - Configure to send a confirmation message to the user  
    - Connect output of "Add your logic here" here

17. **Add Manual Trigger Node "Make a Refund"**  
    - Used to manually initiate refunds  
    - No input configuration needed

18. **Add Set Node "Bot API token (for refund)"**  
    - Enter Telegram Bot API token for refund operations  
    - Connect output of "Make a Refund" here

19. **Add Set Node "Refund Data"**  
    - Prepare refund request data, including user ID and payment charge ID  
    - Connect output of "Bot API token (for refund)" here

20. **Add HTTP Request Node "Refund"**  
    - Configure Telegram API refund method (`refundPayment`) or equivalent endpoint  
    - Use token and refund data prepared  
    - Connect output of "Refund Data" here

21. **Add NoOp Node "Add your Refund logic here"**  
    - Placeholder for any custom refund handling logic or notifications  
    - Connect output of "Actions" third output here or from "Refund" node

22. **Connect all nodes according to the flow described in the connections section**  
    - Ensure correct routing through switches and logic nodes  
    - Test each path individually to confirm behavior

23. **Set Credentials**  
    - Telegram Bot API token: Required in "Bot API token" and "Bot API token (for refund)" Set nodes  
    - Google Sheets credentials: Required for "Write Telegram Payment Charge ID" node  
    - OAuth2 or API key as needed for Google Sheets

24. **Review and customize invoice details**  
    - Product name, description, amount, currency, provider token for payments  
    - Adjust messaging content in Telegram nodes

25. **Enable Telegram Trigger**  
    - Once all nodes are configured and tested, enable the Telegram Trigger node to receive live events

---

### 5. General Notes & Resources

| Note Content                                                                                                  | Context or Link                                                                                     |
|---------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| Telegram Stars are a way for users to support creators via digital payments in Telegram chats.                 | [Telegram Stars Blog](https://telegram.org/blog/telegram-stars?ln=r)                               |
| Setup is very simple and takes about 1 minute; follow the yellow sticky notes for configuration guidance.      | Sticky notes present within the workflow                                                           |
| Workflow supports extensibility for user profile management, payment analytics, CRM and accounting integration. | Can be connected to [Telegram User Registration Workflow](https://n8n.io/workflows/2406-telegram-user-registration-workflow/) |
| Available templates for related workflows: Telegram Bot Starter Template, Telegram User Registration Workflow. | [Starter Template](https://n8n.io/workflows/2402-telegram-bot-starter-template-setup/)             |
| For assistance, reach out to Victor on LinkedIn.                                                               | [Victor Gubanov LinkedIn](https://www.linkedin.com/in/gubanovvictor/)                              |
| The workflow uses Google Sheets for transparent transaction logging.                                           | Make sure Google Sheets API credentials are properly configured                                   |
| Refunds are manually triggered but can be extended with automation as needed.                                  | Manual Trigger node "Make a Refund"                                                                |

---

This documentation covers all workflow nodes, their roles, configurations, and how to rebuild the workflow entirely in n8n. It is designed to support advanced users and AI agents in understanding, reproducing, and extending the Telegram Stars payment and refund workflow.