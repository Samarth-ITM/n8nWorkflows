Automated Post-Purchase Emails for Gumroad with Notion CRM and Gemini AI

https://n8nworkflows.xyz/workflows/automated-post-purchase-emails-for-gumroad-with-notion-crm-and-gemini-ai-11758


# Automated Post-Purchase Emails for Gumroad with Notion CRM and Gemini AI

### 1. Workflow Overview

This workflow automates post-purchase email communication for Gumroad digital product sellers by integrating Gmail, Notion CRM, and AI-generated personalized replies using Google Gemini AI. The primary use case is to enhance customer engagement immediately after a purchase by sending a warm, professional thank-you email while maintaining a customer database to avoid duplicate messaging.

The workflow consists of the following logical blocks:

- **1.1 Input Reception and Parsing**: Watches for new Gumroad purchase emails in Gmail, extracts and normalizes buyer and product data.
- **1.2 Customer Duplication Check**: Queries a Notion CRM database to detect if the customer purchase already exists, preventing duplicate actions.
- **1.3 Customer Record Creation**: Creates a new customer entry in Notion if the purchase is unique.
- **1.4 AI-Driven Email Generation**: Uses an AI agent (powered by Google Gemini) to compose a personalized post-purchase thank-you email.
- **1.5 Email Delivery**: Sends the generated email to the buyer through Gmail.

Supporting the workflow are sticky notes providing setup instructions, context, and detailed explanations for each block.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Parsing

- **Overview:**  
  Listens for new emails in Gmail filtered to detect Gumroad purchase notifications. Extracts structured customer and purchase information using a JavaScript code node.

- **Nodes Involved:**  
  - Gmail Trigger  
  - Code in JavaScript

- **Node Details:**

  - **Gmail Trigger**  
    - Type: Trigger (Gmail)  
    - Role: Watches Gmail inbox for emails matching the query "New sale of " (specific to Gumroad purchase emails) every hour.  
    - Configuration: Uses Gmail OAuth2 credentials; polling set to every hour.  
    - Inputs: None (trigger)  
    - Outputs: New Gmail email messages matching the filter.  
    - Edge cases:  
      - No new emails trigger no downstream execution.  
      - Gmail OAuth2 token expiry or permission issues cause failures.

  - **Code in JavaScript**  
    - Type: Function/Code  
    - Role: Parses raw Gmail email snippet to extract buyer name, email, product, quantity, referrer, purchase date, and generates a unique key.  
    - Configuration: Uses regex patterns to robustly extract fields from email text snippet. Also converts Gmail internalDate to ISO purchase date.  
    - Key expressions: Uses `text.match()` regex functions; constructs a uniqueKey combining email, product, and purchaseDate for deduplication.  
    - Inputs: Gmail email JSON from Gmail Trigger.  
    - Outputs: JSON object with normalized fields for customer and purchase details.  
    - Edge cases:  
      - Emails not matching expected format may yield empty or incorrect fields.  
      - Missing internalDate results in empty purchaseDate.  
      - Regex failures or unexpected email formatting could cause parsing errors.

#### 2.2 Customer Duplication Check

- **Overview:**  
  Checks the Notion database to see if the purchase (based on uniqueKey) already exists to prevent duplicate records and emails.

- **Nodes Involved:**  
  - Get many database pages (Notion)  
  - If  
  - Stop and Error

- **Node Details:**

  - **Get many database pages**  
    - Type: Notion operation (Get All)  
    - Role: Queries the "Gumroad Leads" Notion database filtering for entries where "UniqueKey" equals the parsed uniqueKey from the email.  
    - Configuration: Uses Notion API credentials, manual filter on uniqueKey property.  
    - Inputs: Output JSON from Code in JavaScript node.  
    - Outputs: Array of matching database pages (empty if none).  
    - Edge cases:  
      - Notion API rate limits or credential issues cause failure.  
      - Incorrect database ID or property name breaks filtering.  

  - **If**  
    - Type: Conditional check  
    - Role: Checks if the array of matching pages is empty (length = 0) to determine if purchase is new.  
    - Configuration: Condition on `Object.keys($json).length == 0`.  
    - Inputs: Output from Get many database pages.  
    - Outputs:  
      - True branch: No duplicates found (proceed).  
      - False branch: Duplicate detected (stop).  
    - Edge cases:  
      - Misinterpretation if Notion data structure changes.  

  - **Stop and Error**  
    - Type: Workflow termination with error  
    - Role: Stops the workflow with an error message "The Data already exists" when duplicate purchase is detected.  
    - Inputs: False branch from If node.  
    - Outputs: None (stops flow).  
    - Edge cases:  
      - Workflow stops cleanly on duplicates, no retries.  

#### 2.3 Customer Record Creation

- **Overview:**  
  When a new purchase is confirmed, creates a new page in the Notion "Gumroad Leads" database capturing buyer and purchase details.

- **Nodes Involved:**  
  - Create a database page (Notion)

- **Node Details:**

  - **Create a database page**  
    - Type: Notion operation (Create)  
    - Role: Adds a new record to the Notion database with customer name, email, product, purchase date, quantity, and uniqueKey.  
    - Configuration: Maps JSON fields from the JavaScript node output into respective Notion database properties, uses Notion OAuth2 credentials.  
    - Inputs: True branch from If node.  
    - Outputs: Created Notion page JSON.  
    - Edge cases:  
      - Failure due to invalid credentials or API limits.  
      - Mismatched database schema or property types cause errors.  

#### 2.4 AI-Driven Email Generation

- **Overview:**  
  Generates a personalized, warm, and professional thank-you email body using an AI language model based on the purchase data.

- **Nodes Involved:**  
  - AI Agent  
  - Google Gemini Chat Model (used as language model provider)

- **Node Details:**

  - **AI Agent**  
    - Type: LangChain AI Agent node  
    - Role: Defines a prompt template and sends buyer details to AI model to generate email text.  
    - Configuration: Prompt instructs AI to create a concise, emotionally positive thank-you email including buyer name and product purchased; no subject lines included.  
    - Inputs: Output from Create a database page.  
    - Outputs: JSON containing AI-generated email text in `output` field.  
    - Edge cases:  
      - AI service downtime or quota limits.  
      - Unexpected prompt interpretation leading to inappropriate or empty email content.

  - **Google Gemini Chat Model**  
    - Type: AI language model (Google Gemini)  
    - Role: Provides the AI text generation engine for the AI Agent.  
    - Configuration: Uses Google Gemini credentials (PaLM API).  
    - Inputs: AI prompt from AI Agent node.  
    - Outputs: Generated email text passed back to AI Agent.  
    - Edge cases:  
      - API key expiration or quota exceeded.  
      - Latency or service errors.  

#### 2.5 Email Delivery

- **Overview:**  
  Sends the AI-generated thank-you email to the buyer’s email address via Gmail.

- **Nodes Involved:**  
  - Send a message (Gmail)

- **Node Details:**

  - **Send a message**  
    - Type: Gmail send email node  
    - Role: Sends a plain text email to the customer with a fixed subject, using the AI-generated email body.  
    - Configuration: Recipient is the parsed buyer email from the initial parsing node, subject line fixed as "Thank you for supporting my work ❤️", appending attribution disabled, uses Gmail OAuth2 credentials.  
    - Inputs: Output from AI Agent node.  
    - Outputs: Email send response.  
    - Edge cases:  
      - Invalid email address causes send failure.  
      - Gmail API quota or auth errors.  

---

### 3. Summary Table

| Node Name             | Node Type                           | Functional Role                         | Input Node(s)            | Output Node(s)            | Sticky Note                                                                                   |
|-----------------------|-----------------------------------|---------------------------------------|--------------------------|---------------------------|-----------------------------------------------------------------------------------------------|
| Gmail Trigger         | Gmail Trigger                     | Watches Gmail for new Gumroad emails  | None                     | Code in JavaScript        | Listens for Gumroad purchase emails and extracts buyer and product information. Authenticate Gmail. Ensure Gumroad purchase emails are received in this inbox. |
| Code in JavaScript     | Code Node (JavaScript)             | Parses email text into structured data| Gmail Trigger            | Get many database pages   |                                                                                               |
| Get many database pages| Notion (Get All)                   | Checks for existing purchase duplicates| Code in JavaScript       | If                        | Checks the Notion database for existing customers and stops execution if a duplicate is found. Connect your Notion account. Select the correct customer database. |
| If                    | Conditional                       | Branches workflow on duplicate check  | Get many database pages  | Create a database page (true), Stop and Error (false) |                                                                                               |
| Stop and Error         | Workflow Stop with Error           | Halts workflow on duplicate purchase  | If (false branch)        | None                      | Error when duplicates is detected.                                                           |
| Create a database page | Notion (Create Page)               | Adds customer purchase record          | If (true branch)         | AI Agent                  | Creates a new customer record in Notion for first-time buyers. Stores buyer details, product, value, and timestamp. |
| AI Agent               | LangChain AI Agent                 | Generates personalized thank-you email| Create a database page   | Send a message            | Generates a personalized thank-you email using AI and sends it to the buyer automatically. Instant post-purchase engagement, professional buyer experience, fully automated workflow completion. |
| Google Gemini Chat Model | AI Language Model (Google Gemini) | Provides AI model for email generation | AI Agent (as languageModel) | AI Agent                |                                                                                               |
| Send a message         | Gmail Send Email                   | Sends the generated email to buyer    | AI Agent                 | None                      |                                                                                               |
| Sticky Note            | Sticky Note                       | Notes and instructions                 | None                     | None                      | # Mirai MailFlow automates post-purchase communication for Gumroad creators. Setup requires connecting Gmail, Notion, and AI model. Works on cloud and self-hosted n8n. Setup takes ~5–10 minutes. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Gmail Trigger Node**  
   - Type: Gmail Trigger  
   - Set filter query to: `New sale of ` (to detect Gumroad sale emails)  
   - Poll interval: Every hour  
   - Credentials: Connect Gmail OAuth2 account with read access to inbox  

2. **Add Code Node (JavaScript)**  
   - Type: Code (JavaScript)  
   - Paste the provided robust Gumroad email parser script (as per the workflow)  
   - Input: Connect Gmail Trigger output  
   - Output: JSON with fields: product, name, email, quantity, referrer, purchaseDate, uniqueKey  

3. **Add Notion 'Get Many Database Pages' Node**  
   - Type: Notion (Get All pages)  
   - Connect input from Code node  
   - Configure to query the target Notion database (e.g., "Gumroad Leads")  
   - Filter: UniqueKey equals `{{$json.uniqueKey}}`  
   - Credentials: Connect Notion API OAuth2 account  

4. **Add If Node**  
   - Type: If  
   - Condition: Check if the result from Notion query is empty (`Object.keys($json).length == 0`)  
   - True branch: New purchase detected  
   - False branch: Duplicate purchase detected  

5. **Add Stop and Error Node (on False branch)**  
   - Type: Stop and Error  
   - Error message: "The Data already exists"  
   - Connect input from If node (false branch)  

6. **Add Notion 'Create a Database Page' Node (on True branch)**  
   - Type: Notion (Create Page)  
   - Connect input from If node (true branch)  
   - Map fields from Code node JSON to Notion database properties:  
     - Name → Name (title)  
     - Email → email (rich_text)  
     - Product → Product (rich_text)  
     - Purchase Date → Purchase Date (rich_text)  
     - Quantity → Quantity (rich_text)  
     - UniqueKey → UniqueKey (rich_text)  
   - Credentials: Same Notion API account  

7. **Add AI Agent Node**  
   - Type: LangChain AI Agent  
   - Connect input from Notion Create Page node  
   - Configure prompt:  
     ```
     You are an email writing assistant. Write a short, warm, professional, and emotionally positive email to a customer who has just purchased one of my digital products on Gumroad.
     
     Use the following customer details:
     - Name: {{ $json.name }}
     - Product Purchased: {{ $json.property_product }}
     
     Write the email as if it is coming directly from me (Paul). The tone should be:
     - friendly but professional
     - genuinely appreciative
     - concise (5–7 sentences max)
     - emotionally positive and reassuring
     - no unnecessary marketing fluff
     
     Include:
     - a warm thank-you for their purchase from Gumroad 
     - a sentence appreciating their trust  
     - a brief one-line description of what the product helps with  
     - reassurance that they can reach me anytime for help  
     - a positive closing note wishing them success with the product 
     
     Do NOT include subject lines unless I ask separately.
     Return only the email body in clean paragraph form.
     ```  
   - Use Google Gemini Chat Model as the language model  
   - Credentials: Connect Google PaLM API credentials  

8. **Add Google Gemini Chat Model Node**  
   - Type: AI Language Model (Google Gemini)  
   - Connect to AI Agent node as language model provider  
   - Credentials: Google PaLM API key  

9. **Add Gmail 'Send a Message' Node**  
   - Type: Gmail Send Email  
   - Connect input from AI Agent node output  
   - To: `{{$json.email}}` (buyer email from parsed data)  
   - Subject: "Thank you for supporting my work ❤️"  
   - Message: Use AI-generated email body from AI Agent output (`{{$json.output}}`)  
   - Options: Disable attribution footer  
   - Credentials: Use same Gmail OAuth2 credentials as trigger  

10. **Connect Nodes in the described order:**  
    Gmail Trigger → Code in JavaScript → Get many database pages → If  
    - If True → Create a database page → AI Agent (with Google Gemini) → Send a message  
    - If False → Stop and Error  

11. **Validation and Testing:**  
    - Authenticate all credentials properly (Gmail OAuth2, Notion OAuth2, Google PaLM API)  
    - Ensure Gumroad purchase emails are received in the Gmail inbox monitored  
    - Confirm Notion database schema matches property mappings  
    - Test with sample purchase email to verify parsing, duplication detection, database creation, AI email generation, and email sending  

---

### 5. General Notes & Resources

| Note Content                                                                                                               | Context or Link                                              |
|----------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------|
| # Mirai MailFlow automates post-purchase communication for Gumroad creators. Setup requires connecting Gmail, Notion, and AI model. Works on cloud and self-hosted n8n. Setup takes ~5–10 minutes. | Overview sticky note in workflow                             |
| Listens for Gumroad purchase emails and extracts buyer and product information. Authenticate Gmail. Ensure Gumroad purchase emails are received in this inbox. | Setup instructions for Gmail Trigger                         |
| Checks the Notion database for existing customers and stops execution if a duplicate is found. Connect your Notion account and select the correct customer database. | Setup instructions for Notion duplication check             |
| Creates a new customer record in Notion for first-time buyers. Stores buyer details, product, purchase value, timestamp.   | Setup instructions for Notion database page creation        |
| Generates a personalized thank-you email using AI and sends it automatically to the buyer. Outcome: instant post-purchase engagement, professional buyer experience, fully automated workflow completion. | AI reply and delivery explanation                            |
| [Google PaLM API documentation](https://developers.generativeai.google/tutorials/chat/getting-started)                    | Reference for Google Gemini Chat Model credentials and usage |

---

**Disclaimer:** The provided text is exclusively derived from an automated n8n workflow. It strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All processed data is legal and public.