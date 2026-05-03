Automatically Label Gmail Emails with GPT-5 Classification

https://n8nworkflows.xyz/workflows/automatically-label-gmail-emails-with-gpt-5-classification-7759


# Automatically Label Gmail Emails with GPT-5 Classification

---
### 1. Workflow Overview

This workflow, titled **"Automatically Label Gmail Emails with GPT-5 Classification"**, is designed to automate Gmail inbox organization by leveraging AI-powered email classification. It continuously monitors incoming emails in a Gmail account, classifies each email into predefined categories using OpenAI's GPT-5 and a Langchain text classifier, and automatically applies corresponding Gmail labels. This system is ideal for professionals and teams who handle high volumes of emails and need streamlined prioritization.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Watches Gmail inbox for new messages.
- **1.2 AI Classification:** Uses OpenAI GPT-5 via Langchain nodes to classify email content.
- **1.3 Label Application:** Applies Gmail labels based on classification results.
- **1.4 Documentation & Setup Guide:** A comprehensive sticky note explains usage, setup, and customization.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
This block continuously monitors the Gmail inbox and triggers the workflow whenever new emails arrive.

**Nodes Involved:**  
- Gmail Trigger

**Node Details:**  

- **Gmail Trigger**  
  - **Type & Role:** Event trigger node that polls Gmail for new emails.  
  - **Configuration:** Polls every minute without filters, capturing all incoming emails.  
  - **Expressions/Variables:** None directly used; outputs raw email JSON, including `text` and `id`.  
  - **Connections:** Output connected to the "Text Classifier" node.  
  - **Version Requirements:** Requires Gmail OAuth2 credentials configured. Supports Gmail API v1.3.  
  - **Edge Cases / Failure Modes:** Authentication failures (expired tokens), Gmail API rate limits, network issues causing polling failures.  
  - **Sub-workflow:** None.

---

#### 1.2 AI Classification

**Overview:**  
Processes the raw email text through an OpenAI GPT-5 powered Langchain text classifier to assign one of four categories to each email.

**Nodes Involved:**  
- OpenAI Chat Model  
- Text Classifier

**Node Details:**  

- **OpenAI Chat Model**  
  - **Type & Role:** Calls OpenAI's GPT-5 language model via Langchain to process input text.  
  - **Configuration:** Uses model `"gpt-5"` with default options. API credentials are required.  
  - **Expressions/Variables:** Receives input text dynamically from Gmail Trigger outputs (via Langchain integration).  
  - **Connections:** Output feeds into "Text Classifier" node via ai_languageModel input.  
  - **Version Requirements:** Requires valid OpenAI API key with GPT-5 access. Supports Langchain node v1.2.  
  - **Edge Cases / Failure Modes:** API rate limits, invalid API credentials, network timeouts, model unavailability.  
  - **Sub-workflow:** None.

- **Text Classifier**  
  - **Type & Role:** Langchain node that classifies text into predefined categories based on GPT output.  
  - **Configuration:** Input text is set dynamically as the email body (`{{$json.text}}`). Four categories defined with descriptions and keyword hints:  
    - High Priority  
    - Promotion  
    - Finance/Billing  
    - Customer Support  
  - **Expressions/Variables:** The input text expression dynamically references the Gmail Trigger email text.  
  - **Connections:** Outputs to four separate Gmail nodes (one per category).  
  - **Version Requirements:** Langchain node v1.1 or higher.  
  - **Edge Cases / Failure Modes:** Misclassification due to ambiguous text, empty or malformed email content, expression evaluation errors, API failures inherited from the Chat Model node.  
  - **Sub-workflow:** None.

---

#### 1.3 Label Application

**Overview:**  
Based on the classification category, this block applies the corresponding Gmail label to the email message.

**Nodes Involved:**  
- High Priority  
- Promotion  
- Finance/Billings  
- Customer Support  

**Node Details:**  

- **High Priority (Gmail node)**  
  - **Type & Role:** Adds the "High Priority" Gmail label to the email.  
  - **Configuration:** Adds label with ID `"Label_2050230922550608961"` to message with dynamic message ID `{{$json.id}}`.  
  - **Connections:** Receives input from "Text Classifier" node's "High Priority" output.  
  - **Version Requirements:** Gmail node v2.1 supporting label operations with OAuth2 credentials.  
  - **Edge Cases / Failure Modes:** Invalid or missing label ID, expired OAuth token, Gmail API rate limiting, message ID not found, label sync delays.  
  - **Sub-workflow:** None.

- **Promotion (Gmail node)**  
  - **Type & Role:** Adds promotional labels `"Label_4137435394166770583"` and `"CATEGORY_PROMOTIONS"` to the email.  
  - **Configuration:** Uses message ID from input data; labels correspond to Gmail's Promotions category and custom label.  
  - **Connections:** Receives input from "Text Classifier" node's "Promotion" output.  
  - **Version Requirements:** Same as above.  
  - **Edge Cases / Failure Modes:** Same as above.  
  - **Sub-workflow:** None.

- **Finance/Billings (Gmail node)**  
  - **Type & Role:** Adds the "Finance/Billing" label with ID `"Label_123317652592217245"`.  
  - **Configuration:** Uses dynamic message ID for labeling.  
  - **Connections:** Receives input from "Text Classifier" node's "Finance/Billings" output.  
  - **Version Requirements:** Same as above.  
  - **Edge Cases / Failure Modes:** Same as above.  
  - **Sub-workflow:** None.

- **Customer Support (Gmail node)**  
  - **Type & Role:** Adds the "Customer Support" label with ID `"Label_3866679317291552912"`.  
  - **Configuration:** Uses dynamic message ID for labeling.  
  - **Connections:** Receives input from "Text Classifier" node's "Customer Support" output.  
  - **Version Requirements:** Same as above.  
  - **Edge Cases / Failure Modes:** Same as above.  
  - **Sub-workflow:** None.

---

#### 1.4 Documentation & Setup Guide

**Overview:**  
This sticky note node contains detailed documentation embedded within the workflow, providing users with information about the workflow’s purpose, setup instructions, customization tips, and troubleshooting advice.

**Nodes Involved:**  
- Sticky Note

**Node Details:**  

- **Sticky Note**  
  - **Type & Role:** Informational node with extensive textual content describing the workflow, target users, setup steps, customization options, security considerations, and troubleshooting.  
  - **Configuration:** Static content, no input/output connections.  
  - **Version Requirements:** None specific.  
  - **Edge Cases / Failure Modes:** None.  
  - **Sub-workflow:** None.

---

### 3. Summary Table

| Node Name        | Node Type                                | Functional Role           | Input Node(s)      | Output Node(s)                             | Sticky Note                                                                                                                                                                                                                          |
|------------------|-----------------------------------------|---------------------------|--------------------|--------------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Gmail Trigger    | n8n-nodes-base.gmailTrigger              | Input Reception           | -                  | Text Classifier                            | See sticky note for setup, categories, and customization instructions.                                                                                                                                                            |
| OpenAI Chat Model| @n8n/n8n-nodes-langchain.lmChatOpenAi   | AI Language Model Access  | -                  | Text Classifier (ai_languageModel input)  | See sticky note for AI model usage notes and API considerations.                                                                                                                                                                 |
| Text Classifier  | @n8n/n8n-nodes-langchain.textClassifier | AI Classification         | Gmail Trigger, OpenAI Chat Model | High Priority, Promotion, Finance/Billings, Customer Support | Classification categories and descriptions detailed in sticky note.                                                                                                                                                              |
| High Priority    | n8n-nodes-base.gmail                     | Apply "High Priority" Label | Text Classifier    | -                                          | Labels must be created in Gmail; update label IDs accordingly (see sticky note).                                                                                                                                                  |
| Promotion        | n8n-nodes-base.gmail                     | Apply "Promotion" Label   | Text Classifier    | -                                          | Labels must be created in Gmail; update label IDs accordingly (see sticky note).                                                                                                                                                  |
| Finance/Billings | n8n-nodes-base.gmail                     | Apply "Finance/Billing" Label | Text Classifier    | -                                          | Labels must be created in Gmail; update label IDs accordingly (see sticky note).                                                                                                                                                  |
| Customer Support | n8n-nodes-base.gmail                     | Apply "Customer Support" Label | Text Classifier    | -                                          | Labels must be created in Gmail; update label IDs accordingly (see sticky note).                                                                                                                                                  |
| Sticky Note      | n8n-nodes-base.stickyNote                | Documentation & Setup Guide| -                  | -                                          | Contains full workflow overview, setup, customization, and troubleshooting instructions.                                                                                                                                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Gmail OAuth2 Credential in n8n**  
   - Configure OAuth2 credentials for your Gmail account to allow API access.  
   - Ensure Gmail API scopes include reading emails and modifying labels.

2. **Create OpenAI API Credential in n8n**  
   - Add your OpenAI API key with access to GPT-5 or your preferred GPT model.

3. **Create Gmail Labels**  
   - In your Gmail account, create the following labels (or use existing ones):  
     - High Priority  
     - Promotion  
     - Finance/Billing  
     - Customer Support  
   - Retrieve their label IDs via Gmail API or by testing Gmail nodes in n8n.

4. **Add "Gmail Trigger" Node**  
   - Node Type: `gmailTrigger`  
   - Parameters:  
     - Set polling interval to **every minute**.  
     - No filters by default (optional: add filters if desired).  
   - Credentials: Use Gmail OAuth2 credential.  
   - Position: Place as the first node.

5. **Add "OpenAI Chat Model" Node**  
   - Node Type: `lmChatOpenAi` (Langchain OpenAI Chat)  
   - Parameters:  
     - Model: Select `"gpt-5"` (or alternate model as preferred).  
     - Leave options as default.  
   - Credentials: Use OpenAI API credential.  
   - Position: Downstream node but not connected directly (used as ai_languageModel input).

6. **Add "Text Classifier" Node**  
   - Node Type: `textClassifier` (Langchain)  
   - Parameters:  
     - Input Text: Set to `{{$json.text}}` from Gmail Trigger.  
     - Categories: Define four categories with descriptions and keyword hints:  
       - High Priority  
       - Promotion  
       - Finance/Billing  
       - Customer Support  
   - Connect the ai_languageModel input of this node to the "OpenAI Chat Model" node output.  
   - Connect main input of this node from "Gmail Trigger" output.

7. **Add Four Gmail Nodes for Labeling**  
   For each category, add a Gmail node configured as follows:  
   - **Node Type:** `gmail`  
   - **Parameters:**  
     - Operation: `addLabels`  
     - Message ID: `{{$json.id}}` (from classifier output)  
     - Label IDs: Use the respective Gmail label IDs for each category.  
   - Credentials: Gmail OAuth2 credential.

   Connect each node’s input from the corresponding output of the "Text Classifier" node:  
   - High Priority → High Priority node  
   - Promotion → Promotion node  
   - Finance/Billings → Finance/Billings node  
   - Customer Support → Customer Support node

8. **Add a Sticky Note Node (Optional)**  
   - Paste the detailed workflow description, setup instructions, customization tips, and troubleshooting advice.  
   - Position it clearly for user reference.

9. **Activate Workflow**  
   - Test with sample emails to confirm proper detection, classification, and labeling.  
   - Monitor for API rate limits or errors and adjust polling frequency if necessary.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   | Context or Link                                                                                     |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| This workflow is ideal for professionals, customer support teams, and small business owners overwhelmed by high email volumes. It leverages AI to reduce manual sorting and improve response efficiency.                                                                                                                                                                                                                                                                                                                                                                          | Workflow purpose summary                                                                             |
| To obtain Gmail label IDs, use the Gmail API or test the Gmail node in n8n to list labels. Replace hardcoded label IDs in Gmail nodes accordingly.                                                                                                                                                                                                                                                                                                                                                                                                                              | Gmail label ID retrieval instruction                                                                |
| OpenAI GPT-5 access requires an active API key with sufficient quota. Consider switching to GPT-4 or GPT-3.5-turbo for cost or availability reasons.                                                                                                                                                                                                                                                                                                                                                                                                                             | OpenAI API usage and model selection advice                                                        |
| Recommended to never hardcode API keys directly into the workflow; always use n8n’s credential system for security.                                                                                                                                                                                                                                                                                                                                                                                                                                                             | Security best practice                                                                               |
| For advanced customization, add additional categories, Gmail filters, or actions like starring emails, forwarding, calendar creation, Slack notifications, or ticket creation.                                                                                                                                                                                                                                                                                                                                                                                                | Customization and extension suggestions                                                            |
| Troubleshooting tips include fine-tuning category descriptions, monitoring API and Gmail rate limits, and allowing label sync delays in Gmail clients.                                                                                                                                                                                                                                                                                                                                                                                                                           | Troubleshooting guidance                                                                            |
| Workflow documentation embedded as a sticky note within the workflow node canvas for easy reference.                                                                                                                                                                                                                                                                                                                                                                                                                                                                             | Embedded workflow documentation                                                                     |

---

**Disclaimer:**  
The provided text is exclusively derived from an automated n8n workflow created with the n8n integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All handled data is legal and publicly accessible.