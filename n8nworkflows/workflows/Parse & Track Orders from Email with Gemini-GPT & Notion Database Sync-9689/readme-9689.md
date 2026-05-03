Parse & Track Orders from Email with Gemini/GPT & Notion Database Sync

https://n8nworkflows.xyz/workflows/parse---track-orders-from-email-with-gemini-gpt---notion-database-sync-9689


# Parse & Track Orders from Email with Gemini/GPT & Notion Database Sync

---

## 1. Workflow Overview

This n8n workflow automates the parsing and tracking of e-commerce order emails by extracting structured order data using AI models, then synchronizing that data with a Notion database to maintain updated order records. It also notifies via email upon order status changes. The workflow is designed for e-commerce managers and customer service teams needing automatic ingestion and tracking of order-related emails from various vendors.

The workflow is logically divided into these blocks:

**1.1 Input Reception and Initial Filtering**  
- Triggered by new incoming emails in Gmail.  
- Checks email content to identify if it pertains to an order or shipment.

**1.2 Email Classification and Routing**  
- Uses a custom code node to classify emails as order-related or not.  
- Routes order emails for further processing; non-order emails are discarded.

**1.3 AI Parsing of Order Information**  
- Uses Google Gemini and OpenAI GPT-4 mini chat models for natural language understanding.  
- Parses and extracts structured JSON order data via a structured output parser node.

**1.4 Notion Database Synchronization**  
- Checks if the order exists in Notion by searching with order number.  
- Updates existing order records only upon status changes, or creates new records if none found.  
- Maintains data integrity by preserving immutable fields and tracking status changes with timestamps.

**1.5 Notification and Confirmation**  
- Sends a Gmail notification message upon successful database synchronization.  
- Provides status reports or error messages accordingly.

---

## 2. Block-by-Block Analysis

### 2.1 Input Reception and Initial Filtering

**Overview:**  
This block triggers the workflow on new incoming Gmail messages and prepares the email content for classification.

**Nodes Involved:**  
- Gmail Trigger  
- Check Email Type

**Node Details:**

- **Gmail Trigger**  
  - Type: Email trigger node for Gmail.  
  - Configuration: Polls inbox every minute for new emails without filters.  
  - Inputs: None (trigger).  
  - Outputs: JSON containing email metadata and body.  
  - Potential Failures: OAuth token expiration, Gmail API limits.

- **Check Email Type**  
  - Type: Code node (JavaScript).  
  - Role: Analyzes email subject and body for order-related keywords/patterns.  
  - Key Logic: Uses regex and keyword lists to detect order indicators, shipping terms, and order phrases.  
  - Outputs: `isOrderEmail` (boolean), `confidence` (high/medium/low), matched indicators, and debug preview.  
  - Inputs: Email JSON from Gmail Trigger.  
  - Outputs: Classification JSON.  
  - Edge Cases: False negatives if email lacks keywords; false positives if unrelated content contains keywords.

---

### 2.2 Email Classification and Routing

**Overview:**  
Routes emails based on classification result to either process order emails or discard non-order emails.

**Nodes Involved:**  
- Email Router  
- No action taken

**Node Details:**

- **Email Router**  
  - Type: If node.  
  - Role: Routes flow based on `isOrderEmail` boolean from previous node.  
  - True branch: Proceeds to AI extraction.  
  - False branch: Ends workflow (No action taken).  
  - Inputs: Classification JSON.  
  - Outputs: Routed branches.  
  - Edge Cases: Misclassification leads to lost orders or unnecessary processing.

- **No action taken**  
  - Type: NoOp (no operation) node.  
  - Role: Terminates workflow for non-order emails.  
  - Inputs: From false branch of Email Router.  
  - Outputs: None.

---

### 2.3 AI Parsing of Order Information

**Overview:**  
This block uses Google Gemini and OpenAI GPT models to extract structured order information from email content, standardized into JSON via a structured output parser.

**Nodes Involved:**  
- Google Gemini Chat Model  
- OpenAI Chat Model  
- Structured Output Parser  
- Email Classification and Extraction Agent

**Node Details:**

- **Google Gemini Chat Model**  
  - Type: Langchain Google Gemini chat model node.  
  - Configuration: Uses "models/gemini-2.5-pro" model with default options.  
  - Credentials: Google Palm API account.  
  - Role: Processes email text to extract order data.  
  - Inputs: Email content from classification.  
  - Outputs: Model response with order data.  
  - Edge Cases: API rate limits, model errors, incomplete extraction.

- **OpenAI Chat Model**  
  - Type: Langchain OpenAI GPT chat node.  
  - Configuration: Uses GPT-4.1-mini model, default options.  
  - Credentials: OpenAI API key.  
  - Role: Alternative or parallel model for order extraction.  
  - Inputs: Same email content.  
  - Outputs: Model response with order data.  
  - Edge Cases: Same as Gemini node.

- **Structured Output Parser**  
  - Type: Langchain structured output parser.  
  - Configuration: Uses a JSON schema example defining required fields such as order number, items, status, delivery info, and confidence.  
  - Role: Parses AI model outputs into validated JSON structure matching the schema.  
  - Inputs: AI model responses.  
  - Outputs: Clean structured JSON order data.  
  - Edge Cases: Parsing failure if AI output deviates from schema.

- **Email Classification and Extraction Agent**  
  - Type: Langchain agent node.  
  - Role: Orchestrates AI extraction using system prompt describing extraction rules, supported vendors, data fields, and output format.  
  - Inputs: Email text combined subject and snippet.  
  - Outputs: Final extracted JSON order data.  
  - Edge Cases: Ambiguous or incomplete emails may reduce extraction confidence.

---

### 2.4 Notion Database Synchronization

**Overview:**  
Synchronizes extracted order data with a Notion database: searches existing orders, updates if status changed, or creates new order entries. Ensures data integrity and prevents duplicates.

**Nodes Involved:**  
- Search a page in Notion  
- Order Database Sync Agent  
- Create a database page in Notion  
- Update a database page in Notion  
- No action taken (sometimes used to terminate no-update cases)

**Node Details:**

- **Search a page in Notion**  
  - Type: Notion database search tool.  
  - Configuration: Searches Notion database by exact order number (title property).  
  - Credentials: Personal Notion API account.  
  - Role: Determines if order exists to decide update or create.  
  - Inputs: Order number from extracted data.  
  - Outputs: Search results array.  
  - Edge Cases: API errors treated as “order not found” to trigger create.

- **Order Database Sync Agent**  
  - Type: Langchain agent node.  
  - Role: Implements complex logic for:  
    - Input validation (ensuring required fields present)  
    - Search response evaluation (no response, empty, single, multiple results)  
    - Update existing order if status changed, preserving original immutable fields  
    - Create new order if none found  
    - Duplicate detection and error reporting  
    - Status regression warnings and timestamped notes  
    - Outputting human-readable status confirmation messages  
  - Inputs: Structured JSON from extraction agent, Notion search results.  
  - Outputs: Action commands and confirmation messages.  
  - Edge Cases: Missing required fields abort with validation error; duplicates require manual intervention; connection errors report failure; backward status changes flagged.

- **Create a database page in Notion**  
  - Type: Notion create page tool.  
  - Configuration: Creates new order with required properties (Order Number, Item Name, Quantity, Expected Date, Status) and optional properties (Vendor, Customer Name, Price, Order Total, Currency, Delivery Location, Notes).  
  - Credentials: Personal Notion API account.  
  - Inputs: Data from sync agent.  
  - Outputs: Created page metadata.  
  - Edge Cases: API errors on creation; missing optional fields handled gracefully.

- **Update a database page in Notion**  
  - Type: Notion update page tool.  
  - Configuration: Updates order status, expected delivery, notes with timestamp; preserves immutable fields.  
  - Credentials: Personal Notion API account.  
  - Inputs: Page ID, updated properties from sync agent.  
  - Outputs: Updated page metadata.  
  - Edge Cases: API errors; no update if status unchanged; handles status regression warnings.

- **No action taken** (reused)  
  - Used to terminate paths where no update is needed.

---

### 2.5 Notification and Confirmation

**Overview:**  
Sends email notifications summarizing order updates or creation status to a designated recipient.

**Nodes Involved:**  
- Send a message

**Node Details:**

- **Send a message**  
  - Type: Gmail send email node.  
  - Configuration: Sends plain text email with subject “ORDER CHANGE” to fixed recipient (`kiasddsadd@gmail.com`).  
  - Inputs: Confirmation message from Order Database Sync Agent.  
  - Credentials: Gmail OAuth2 account.  
  - Edge Cases: Email delivery failures, OAuth token expiration.

---

## 3. Summary Table

| Node Name                       | Node Type                           | Functional Role                                 | Input Node(s)                     | Output Node(s)                       | Sticky Note                                                                                             |
|--------------------------------|-----------------------------------|------------------------------------------------|----------------------------------|------------------------------------|-------------------------------------------------------------------------------------------------------|
| Gmail Trigger                  | n8n-nodes-base.gmailTrigger        | Triggers workflow on new Gmail message          | None                             | Check Email Type                    | ## Email Triger - The workflow is triggered when a new email is received in the inbox                  |
| Check Email Type               | n8n-nodes-base.code                | Classifies email content as order-related or not| Gmail Trigger                   | Email Router                       | **Email Classification Node** - Analyzes incoming emails for order/shipment content using patterns    |
| Email Router                  | n8n-nodes-base.if                  | Routes flow based on order email classification  | Check Email Type                 | Email Classification and Extraction Agent, No action taken | **Order Email Router** - Routes order emails for processing, discards non-orders                        |
| No action taken               | n8n-nodes-base.noOp                | Terminates workflow for non-order emails         | Email Router (false branch)      | None                              | ## Take No action - This branch handles non-order emails by terminating                                |
| Google Gemini Chat Model      | @n8n/n8n-nodes-langchain.lmChatGoogleGemini | Extracts order data using Google's Gemini model | Email Classification and Extraction Agent | Order Database Sync Agent, Email Classification and Extraction Agent (ai_languageModel outputs) |                                                                                                       |
| OpenAI Chat Model             | @n8n/n8n-nodes-langchain.lmChatOpenAi | Alternative AI model for order data extraction   | Email Classification and Extraction Agent | Order Database Sync Agent, Email Classification and Extraction Agent (ai_languageModel outputs) |                                                                                                       |
| Structured Output Parser      | @n8n/n8n-nodes-langchain.outputParserStructured | Parses AI model output into structured JSON      | AI models (Gemini/OpenAI)        | Email Classification and Extraction Agent |                                                                                                       |
| Email Classification and Extraction Agent | @n8n/n8n-nodes-langchain.agent | Orchestrates AI extraction of order data         | Email Router (true branch)       | Order Database Sync Agent           | **Email Order Extraction Agent** - AI-powered parser extracts structured order data from emails       |
| Order Database Sync Agent     | @n8n/n8n-nodes-langchain.agent    | Validates data, searches Notion, creates/updates orders | Email Classification and Extraction Agent, Search a page in Notion, Create/Update Notion nodes | Send a message                     | **Notion Order Database Sync Agent** - Intelligent database manager creating/updating Notion orders   |
| Search a page in Notion       | n8n-nodes-base.notionTool          | Searches Notion DB for existing order by number | Order Database Sync Agent        | Order Database Sync Agent           | **Notion Integration Tools** - Search, Create, Update database pages with authenticated calls          |
| Create a database page in Notion | n8n-nodes-base.notionTool        | Creates new order record in Notion database      | Order Database Sync Agent        | Order Database Sync Agent           |                                                                                                       |
| Update a database page in Notion | n8n-nodes-base.notionTool        | Updates existing order record in Notion           | Order Database Sync Agent        | Order Database Sync Agent           |                                                                                                       |
| Send a message               | n8n-nodes-base.gmail              | Sends notification email about order status      | Order Database Sync Agent        | None                              |                                                                                                       |

---

## 4. Reproducing the Workflow from Scratch

1. **Create Gmail Trigger**  
   - Type: Gmail Trigger  
   - Credentials: Connect with Gmail OAuth2 account  
   - Settings: Poll every minute, no filters  
   - Position: Start node

2. **Add Code Node: Check Email Type**  
   - Type: Code (JavaScript)  
   - Input: Output from Gmail Trigger  
   - Paste the provided JavaScript code to analyze email subject/body for order-related keywords  
   - Outputs: JSON with `isOrderEmail` boolean and confidence levels

3. **Add If Node: Email Router**  
   - Condition: `isOrderEmail` equals `true` (boolean)  
   - True branch: proceed to AI extraction  
   - False branch: connect to No action taken node

4. **Add NoOp Node: No action taken**  
   - Connect from Email Router false branch  
   - Terminates processing of non-order emails

5. **Add AI Model Nodes: Google Gemini Chat Model & OpenAI Chat Model**  
   - Gemini Node: Configure with "models/gemini-2.5-pro" and Google Palm API credentials  
   - OpenAI Node: Configure with GPT-4.1-mini model and OpenAI API credentials  
   - Input: Email content (subject + body) from Email Router true branch  
   - Both models run parallel extraction

6. **Add Structured Output Parser Node**  
   - Configure with JSON schema example defining order data fields  
   - Input: AI models output  
   - Output: Validated structured JSON of order data

7. **Add Email Classification and Extraction Agent**  
   - Configure with a detailed prompt describing extraction roles, requirements, and output format (per provided prompt)  
   - Input: Combined email content + AI model outputs + structured parser output  
   - Output: Clean JSON order data for downstream use

8. **Add Notion Search Node: Search a page in Notion**  
   - Credentials: Personal Notion account  
   - Database ID: Use your Notion order management database ID  
   - Search: By property "Order Number" (title) exact match with order number from extraction agent output

9. **Add Order Database Sync Agent**  
   - Configure with detailed logic prompt for validating input, searching database, deciding create/update, handling duplicates, status comparisons, notes timestamps, and outputting confirmation messages  
   - Inputs: Extracted order data and Notion search results  
   - Outputs: Commands for create/update nodes, success or error messages

10. **Add Notion Create Page Node**  
    - Credentials: Personal Notion account  
    - Database ID: Same as search node  
    - Properties: Map required and optional order fields from sync agent output

11. **Add Notion Update Page Node**  
    - Credentials: Personal Notion account  
    - Input: Page ID from search results  
    - Properties: Update only status, expected delivery date, notes with timestamp (per sync agent instructions)

12. **Add Gmail Send Node: Send a message**  
    - Credentials: Gmail OAuth2 account  
    - To: Fixed recipient email address for notifications  
    - Subject: "ORDER CHANGE"  
    - Message: Use confirmation textual output from sync agent

13. **Connect Nodes**  
    - Gmail Trigger → Check Email Type → Email Router  
    - Email Router (true) → AI Models (Gemini & OpenAI) → Structured Output Parser → Email Classification and Extraction Agent → Order Database Sync Agent  
    - Order Database Sync Agent → Notion Search / Create / Update nodes accordingly  
    - Order Database Sync Agent → Send a message  
    - Email Router (false) → No action taken

14. **Test and Debug**  
    - Test with sample order emails from various vendors  
    - Verify classification accuracy, data extraction, Notion synchronization, and notification delivery

---

## 5. General Notes & Resources

| Note Content                                                                                                         | Context or Link                                                                                          |
|----------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------|
| Workflow triggered automatically on new Gmail messages, enabling near real-time order tracking.                      | Gmail Trigger sticky note                                                                                |
| Email classification uses multi-tier detection: regex patterns, keywords, order phrases to reduce false positives.   | Email Classification sticky note                                                                         |
| AI extraction agents use detailed prompt engineering to ensure vendor-agnostic and standardized order data output.   | Email Order Extraction Agent sticky note                                                                |
| Notion synchronization handles duplicates, status regressions, preserves immutable fields, and gracefully handles errors. | Notion Order Database Sync Agent sticky note                                                            |
| Notion API native nodes are used for database search, creation, and update, ensuring secure, authenticated access.   | Notion Integration Tools sticky note                                                                     |
| Non-order emails are terminated early to optimize workflow efficiency.                                               | Take No Action sticky note                                                                               |
| Status values strictly normalized to one of: Ordered, Shipped, Out for Delivery, Delivered.                         | Extraction and sync agents prompt instructions                                                          |
| Timestamp format for notes uses UAE timezone (GMT+4) to maintain consistent logs.                                    | Sync agent detailed instructions                                                                        |
| Notification emails sent for every create or status change to allow monitoring and manual intervention if needed.   | Send a message node                                                                                       |

---

**Disclaimer:**  
The provided text originates exclusively from an automated n8n workflow designed for order email parsing and synchronization. All handling complies with content policies and processes only legal, public data.

---