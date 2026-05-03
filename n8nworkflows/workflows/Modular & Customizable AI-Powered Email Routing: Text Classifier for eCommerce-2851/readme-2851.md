Modular & Customizable AI-Powered Email Routing: Text Classifier for eCommerce

https://n8nworkflows.xyz/workflows/modular---customizable-ai-powered-email-routing--text-classifier-for-ecommerce-2851


# Modular & Customizable AI-Powered Email Routing: Text Classifier for eCommerce

### 1. Workflow Overview

This workflow automates the processing of contact form submissions for eCommerce businesses by leveraging AI-powered text classification to route emails and log data efficiently. It is designed to classify incoming messages into predefined categories and send them to the appropriate department, while simultaneously logging all interactions in Google Sheets for tracking and analysis.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception**: Captures user inputs from a contact form submission.
- **1.2 AI Text Classification**: Uses an OpenAI GPT-4 model to classify the message into one of several categories.
- **1.3 Email Routing**: Sends the classified message to the relevant department via email.
- **1.4 Data Logging**: Appends submission and classification data to department-specific Google Sheets.
- **1.5 AI Model Integration**: Provides the AI language model used for classification.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview**: This block captures user data from a contact form submission, including name, email, and message, and triggers the workflow.
- **Nodes Involved**: 
  - On form submission

- **Node Details**:

  - **On form submission**
    - **Type & Role**: `formTrigger` node; triggers workflow on form submission.
    - **Configuration**: 
      - Form titled "Contacts" with three required fields: Name (text), Email (text), and Message (textarea).
      - Response mode set to "lastNode" to return the final node's output.
      - Form description: "Basic Contact Form".
      - Webhook ID configured for external form integration.
    - **Expressions/Variables**: Outputs JSON with fields: Name, Email, Message.
    - **Connections**: Output connected to "Text Classifier".
    - **Version Requirements**: Uses typeVersion 2.2, requires n8n version supporting formTrigger node.
    - **Potential Failures**: 
      - Missing required fields if form validation fails.
      - Webhook connectivity issues if external form not properly configured.
      - Data format inconsistencies if form fields are altered externally.

#### 1.2 AI Text Classification

- **Overview**: Classifies the submitted message into one of the predefined categories using GPT-4, enabling targeted routing.
- **Nodes Involved**: 
  - Text Classifier
  - OpenAI (language model provider)

- **Node Details**:

  - **Text Classifier**
    - **Type & Role**: `@n8n/n8n-nodes-langchain.textClassifier`; performs AI-based text classification.
    - **Configuration**: 
      - Uses a system prompt instructing classification into categories: Request Quote, Product info, General problem, Order.
      - Fallback category set to "other".
      - Input text dynamically set to the submitted message (`{{$json.Message}}`).
      - Outputs classification result as JSON with the selected category.
    - **Expressions/Variables**: Input text from form submission; categories hardcoded in node parameters.
    - **Connections**: Receives input from "On form submission"; outputs to multiple email nodes based on classification.
    - **Version Requirements**: Requires n8n version supporting LangChain nodes; typeVersion 1.
    - **Potential Failures**: 
      - OpenAI API errors (rate limits, auth failures).
      - Misclassification if input text is ambiguous.
      - Expression errors if input JSON structure changes.
    - **Sub-workflow**: None.

  - **OpenAI**
    - **Type & Role**: `@n8n/n8n-nodes-langchain.lmChatOpenAi`; provides GPT-4 model for classification.
    - **Configuration**: 
      - Model set to "gpt-4o-mini".
      - No additional options set.
      - Credentials configured with OpenAI API key.
    - **Expressions/Variables**: None directly; serves as AI backend.
    - **Connections**: Connected as AI language model provider to "Text Classifier".
    - **Version Requirements**: Requires OpenAI API credentials and n8n version supporting LangChain OpenAI node.
    - **Potential Failures**: 
      - API key invalid or expired.
      - Network or timeout errors.
      - Model unavailability or quota exceeded.

#### 1.3 Email Routing

- **Overview**: Routes the classified message to the appropriate department by sending an email with user details and classification.
- **Nodes Involved**: 
  - Prod. Dep.
  - Quote Dep.
  - Gen. Dep.
  - Order Dep.
  - Other Dep.

- **Node Details**:

  Each node is an `emailSend` node configured to send emails to a specific department:

  - **Common Configuration**:
    - Email body includes: Name, Email, Message, and a field "Tipo prodotto" (likely a placeholder or legacy field).
    - Reply-To header set dynamically to the user's email.
    - Subject line prefixed with "[n8n Contacts]" followed by the department name.
    - To and From email addresses configured (placeholders "to@domain.com" and "from@domain.com").
    - SMTP credentials configured with a shared SMTP account.
    - TypeVersion 2.1.

  - **Individual Nodes**:
    - **Prod. Dep.**: Subject "[n8n Contacts] Product info".
    - **Quote Dep.**: Subject "[n8n Contacts] Quote".
    - **Gen. Dep.**: Subject "[n8n Contacts] General".
    - **Order Dep.**: Subject "[n8n Contacts] Order info".
    - **Other Dep.**: Subject "[n8n Contacts] Other".

  - **Connections**:
    - Each email node receives input from "Text Classifier" on specific output branches corresponding to classification categories.
    - Each email node outputs to its respective Google Sheets logging node.

  - **Potential Failures**:
    - SMTP authentication errors.
    - Email sending failures due to invalid addresses or server issues.
    - Missing or malformed email content if input data is incomplete.
    - Reply-To header issues if user email is invalid.

#### 1.4 Data Logging

- **Overview**: Logs each submission and classification result into department-specific Google Sheets for record-keeping and analysis.
- **Nodes Involved**: 
  - Prod DB
  - Quote DB
  - General DB
  - Order DB
  - Other DB

- **Node Details**:

  Each node is a `googleSheets` node configured to append a row to a specific sheet within the same Google Sheets document:

  - **Common Configuration**:
    - Operation: Append.
    - Document ID: Shared Google Sheets document.
    - Sheet Name: "gid=0" (likely the default sheet or a placeholder).
    - Columns mapped:
      - TO: JSON stringified email recipients.
      - DATA: Submission timestamp from "Text Classifier" node.
      - NOME: User's name.
      - EMAIL: User's email.
      - CATEGORIA: Hardcoded as "info prodotti" (likely a placeholder; ideally should reflect actual category).
      - RICHIESTA: User's message.
    - Credentials: Google Sheets OAuth2 account.
    - TypeVersion 4.5.

  - **Connections**:
    - Each Google Sheets node receives input from its corresponding email node.

  - **Potential Failures**:
    - Google Sheets API authentication or permission errors.
    - Incorrect or missing sheet names causing append failures.
    - Data mapping errors if input JSON structure changes.
    - API rate limits or quota exceeded.

#### 1.5 AI Model Integration

- **Overview**: Provides the GPT-4 AI model used by the Text Classifier node to perform text classification.
- **Nodes Involved**: 
  - OpenAI (already detailed in 1.2)

- **Node Details**: See 1.2.

---

### 3. Summary Table

| Node Name          | Node Type                          | Functional Role                  | Input Node(s)          | Output Node(s)                        | Sticky Note                                                                                                      |
|--------------------|----------------------------------|--------------------------------|-----------------------|-------------------------------------|------------------------------------------------------------------------------------------------------------------|
| On form submission  | formTrigger                      | Captures form submission input | -                     | Text Classifier                     | This very simple workflow is ideal for eCommerce businesses or customer support teams looking to automate and streamline the handling of contact form submissions. It is possible to hook any external form such as CF7 for Wordpress through a webhook. It is possible to send the email through other providers by replacing them with the relative nodes (Gmail, Outlook....). It is possible to change the collection database with other tools. |
| Text Classifier     | LangChain textClassifier         | Classifies message text         | On form submission     | Quote Dep., Prod. Dep., Gen. Dep., Order Dep., Other Dep. |                                                                                                                  |
| OpenAI             | LangChain lmChatOpenAi            | Provides GPT-4 model            | -                     | Text Classifier (AI language model) |                                                                                                                  |
| Prod. Dep.          | emailSend                       | Sends product info emails       | Text Classifier        | Prod DB                            |                                                                                                                  |
| Quote Dep.          | emailSend                       | Sends quote request emails      | Text Classifier        | Quote DB                          |                                                                                                                  |
| Gen. Dep.           | emailSend                       | Sends general problem emails    | Text Classifier        | General DB                        |                                                                                                                  |
| Order Dep.          | emailSend                       | Sends order-related emails      | Text Classifier        | Order DB                         |                                                                                                                  |
| Other Dep.          | emailSend                       | Sends other category emails     | Text Classifier        | Other DB                         |                                                                                                                  |
| Prod DB             | googleSheets                   | Logs product info submissions   | Prod. Dep.             | -                                 |                                                                                                                  |
| Quote DB            | googleSheets                   | Logs quote request submissions  | Quote Dep.             | -                                 |                                                                                                                  |
| General DB          | googleSheets                   | Logs general problem submissions| Gen. Dep.              | -                                 |                                                                                                                  |
| Order DB            | googleSheets                   | Logs order-related submissions  | Order Dep.             | -                                 |                                                                                                                  |
| Other DB            | googleSheets                   | Logs other category submissions | Other Dep.             | -                                 |                                                                                                                  |
| Sticky Note         | stickyNote                     | Notes and instructions          | -                     | -                                 | This very simple workflow is ideal for eCommerce businesses or customer support teams looking to automate and streamline the handling of contact form submissions. It is possible to hook any external form such as CF7 for Wordpress through a webhook. It is possible to send the email through other providers by replacing them with the relative nodes (Gmail, Outlook....). It is possible to change the collection database with other tools. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Form Trigger Node**
   - Add a `formTrigger` node named "On form submission".
   - Configure form title as "Contacts".
   - Add three required fields:
     - Name (text)
     - Email (text)
     - Message (textarea)
   - Set response mode to "lastNode".
   - Add a webhook ID (auto-generated or custom).
   - Save.

2. **Add the Text Classifier Node**
   - Add a `@n8n/n8n-nodes-langchain.textClassifier` node named "Text Classifier".
   - Set input text to `={{ $json.Message }}`.
   - Define categories:
     - Request Quote: Request for quote
     - Product info: General information about a product
     - General problem: General problems about a product
     - Order: Information about an order placed
   - Set fallback category to "other".
   - Configure system prompt template:  
     `Please classify the text provided by the user into one of the following categories: {categories}, and use the provided formatting instructions below. Don't explain, and only output the json with the selected {categories}.`
   - Connect "On form submission" output to this node's input.

3. **Add the OpenAI Node**
   - Add a `@n8n/n8n-nodes-langchain.lmChatOpenAi` node named "OpenAI".
   - Select model "gpt-4o-mini".
   - Configure OpenAI API credentials.
   - Connect this node as the AI language model provider for the "Text Classifier" node.

4. **Add Email Send Nodes for Each Department**
   - Add five `emailSend` nodes named:
     - Prod. Dep.
     - Quote Dep.
     - Gen. Dep.
     - Order Dep.
     - Other Dep.
   - For each node:
     - Set "To Email" to the department's email address.
     - Set "From Email" to the sender address.
     - Set "Reply-To" to `={{ $json.Email }}`.
     - Configure subject line accordingly:
       - Prod. Dep.: "[n8n Contacts] Product info"
       - Quote Dep.: "[n8n Contacts] Quote"
       - Gen. Dep.: "[n8n Contacts] General"
       - Order Dep.: "[n8n Contacts] Order info"
       - Other Dep.: "[n8n Contacts] Other"
     - Email body (HTML) template:  
       ```
       Name: {{ $json.Name }}
       Email: {{ $json.Email }}

       Message:
       {{ $json.Message }}

       Tipo prodotto: {{ $json["tipo prodotto"] }}
       ```
     - Configure SMTP credentials for sending emails.
   - Connect "Text Classifier" outputs to these nodes based on classification:
     - Output 0: Quote Dep.
     - Output 1: Prod. Dep.
     - Output 2: Gen. Dep.
     - Output 3: Order Dep.
     - Output 4: Other Dep.

5. **Add Google Sheets Nodes for Logging**
   - Add five `googleSheets` nodes named:
     - Quote DB
     - Prod DB
     - General DB
     - Order DB
     - Other DB
   - For each node:
     - Set operation to "append".
     - Set document ID to your Google Sheets document.
     - Set sheet name to the appropriate sheet (e.g., "gid=0" or specific sheet).
     - Map columns:
       - TO: `={{ (JSON.stringify($json.envelope.to)) }}`
       - DATA: `={{ $('Text Classifier').item.json.submittedAt }}`
       - NOME: `={{ $('Text Classifier').item.json.Name }}`
       - EMAIL: `={{ $('Text Classifier').item.json.Email }}`
       - CATEGORIA: (set appropriately; currently hardcoded as "info prodotti" but should reflect actual category)
       - RICHIESTA: `={{ $('Text Classifier').item.json.Message }}`
     - Configure Google Sheets OAuth2 credentials.
   - Connect each email node output to its corresponding Google Sheets node.

6. **Add a Sticky Note (Optional)**
   - Add a `stickyNote` node with notes about workflow usage and customization options.

7. **Test the Workflow**
   - Submit a test form with sample data.
   - Verify classification accuracy.
   - Confirm emails are sent to correct departments.
   - Check Google Sheets for logged data.

8. **Activate the Workflow**
   - Once verified, activate the workflow to enable automation.

---

### 5. General Notes & Resources

| Note Content                                                                                               | Context or Link                                                                                      |
|------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| This workflow is ideal for eCommerce businesses or customer support teams looking to automate contact form handling. | General workflow purpose.                                                                           |
| It is possible to hook any external form such as Contact Form 7 (CF7) for WordPress through the webhook.   | Integration flexibility.                                                                            |
| Email sending can be replaced with other providers like Gmail or Outlook by swapping the email nodes.      | Customization of email delivery.                                                                   |
| The data logging can be adapted to other databases or tools instead of Google Sheets.                      | Data storage customization.                                                                        |
| OpenAI GPT-4 model is used for classification; ensure API credentials and quotas are properly managed.     | AI integration requirements.                                                                       |
| Google Sheets document used for logging must have appropriate sheets and permissions configured.           | Google Sheets setup.                                                                               |

---

This documentation provides a comprehensive understanding of the workflow structure, node configurations, and reproduction steps, enabling advanced users and AI agents to maintain, extend, or adapt the workflow effectively.