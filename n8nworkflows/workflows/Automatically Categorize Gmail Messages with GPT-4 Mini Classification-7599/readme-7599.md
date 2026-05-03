Automatically Categorize Gmail Messages with GPT-4 Mini Classification

https://n8nworkflows.xyz/workflows/automatically-categorize-gmail-messages-with-gpt-4-mini-classification-7599


# Automatically Categorize Gmail Messages with GPT-4 Mini Classification

### 1. Workflow Overview

This workflow automatically categorizes incoming Gmail messages into predefined categories using OpenAI’s GPT-4 Mini model via LangChain’s text classification node. The primary use case is to organize emails into labels like Business, Sales (cold emails), Meetings, or Random, improving email management and productivity.

The workflow consists of the following logical blocks:

- **1.1 Input Reception:** Watches for new Gmail messages using a Gmail Trigger node.
- **1.2 AI-Based Text Classification:** Uses LangChain’s Text Classifier with GPT-4 Mini to categorize the email based on the subject and snippet.
- **1.3 Email Labeling and Management:** Applies Gmail labels or deletes emails based on the classification result.
- **1.4 No-Op Fallback:** A no-operation node to handle emails not fitting predefined categories.
- **1.5 Credentials & Documentation:** Setup guidance for OpenAI and Gmail credentials.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
This block listens for new incoming emails in Gmail, polling every minute, and passes the email data downstream for classification.

**Nodes Involved:**  
- Gmail Trigger

**Node Details:**

- **Gmail Trigger**  
  - *Type:* n8n-nodes-base.gmailTrigger  
  - *Role:* Watches Gmail inbox for new emails.  
  - *Configuration:* Polls every minute with no additional filters, retrieving new email threads.  
  - *Key Variables:* Outputs full JSON of incoming email thread including `id`, `Subject`, and `snippet`.  
  - *Input:* None (trigger node).  
  - *Output:* Passes email data to the “Text Classifier” node.  
  - *Version:* 1.3  
  - *Potential Failures:* Authentication errors (OAuth token expiry), API rate limits, network issues.

---

#### 1.2 AI-Based Text Classification

**Overview:**  
Classifies the incoming email into one of four categories (Business, Sales, Meetings, Random) based on subject and snippet using OpenAI GPT-4 Mini via LangChain’s Text Classifier.

**Nodes Involved:**  
- Text Classifier  
- OpenAI Chat Model (used internally by Text Classifier)

**Node Details:**

- **Text Classifier**  
  - *Type:* @n8n/n8n-nodes-langchain.textClassifier  
  - *Role:* Uses an AI model to classify input text into predefined categories.  
  - *Configuration:*  
    - Input text is constructed by combining the email's subject and snippet:  
      `subject: {{ $json.Subject }}\nMessage: {{ $json.snippet }}`  
    - Categories defined are:  
      - Business: Work-related emails from friends or coworkers  
      - Sales: Cold emails attempting to sell something  
      - Meetings: Emails mentioning meetings, schedules, or reminders  
      - Random: Emails that don’t fit other categories  
  - *Input:* Receives email data from Gmail Trigger.  
  - *Output:* Four outputs mapped respectively to the four categories.  
  - *Version:* 1.1  
  - *Potential Failures:* Expression evaluation errors if JSON fields missing; API timeouts or quota limits; misclassification due to ambiguous input.  
  - *Sub-workflow:* Invokes “OpenAI Chat Model” to perform the classification.

- **OpenAI Chat Model**  
  - *Type:* @n8n/n8n-nodes-langchain.lmChatOpenAi  
  - *Role:* Provides GPT-4 Mini model inference for classification.  
  - *Configuration:* Uses GPT-4 Mini (`gpt-4.1-mini`) model with no extra options.  
  - *Input:* Receives classification prompt from Text Classifier via LangChain integration.  
  - *Output:* Returns classification results to Text Classifier.  
  - *Credentials:* Uses OpenAI API credentials (Hostinger-hosted).  
  - *Version:* 1.2  
  - *Potential Failures:* API key invalidation, rate limits, network errors.

---

#### 1.3 Email Labeling and Management

**Overview:**  
Based on the classification output, applies Gmail labels or deletes the email thread accordingly.

**Nodes Involved:**  
- Business (Gmail node)  
- Cold Emails (Gmail node)  
- Meetings (Gmail node)  
- No Operation, do nothing

**Node Details:**

- **Business**  
  - *Type:* n8n-nodes-base.gmail  
  - *Role:* Adds the "IMPORTANT" Gmail label to threads classified as Business.  
  - *Configuration:*  
    - Operation: Add label to thread  
    - Label ID: “IMPORTANT”  
    - Thread ID is dynamically extracted from the Gmail Trigger node (`{{ $('Gmail Trigger').item.json.id }}`)  
  - *Input:* From the Business classification output of Text Classifier.  
  - *Output:* None (end node).  
  - *Version:* 2.1  
  - *Potential Failures:* Label ID invalid or missing; thread ID missing; auth errors.

- **Cold Emails**  
  - *Type:* n8n-nodes-base.gmail  
  - *Role:* Deletes email threads classified as Sales (cold emails).  
  - *Configuration:*  
    - Operation: Delete Gmail thread  
    - Thread ID from classification output (`{{ $json.id }}`)  
  - *Input:* From Sales classification output of Text Classifier.  
  - *Output:* None.  
  - *Version:* 2.1  
  - *Potential Failures:* Thread already deleted; insufficient permissions; auth errors.

- **Meetings**  
  - *Type:* n8n-nodes-base.gmail  
  - *Role:* Adds the "YELLOW_STAR" Gmail label to threads classified as Meetings.  
  - *Configuration:*  
    - Operation: Add label to thread  
    - Label ID: “YELLOW_STAR”  
    - Thread ID from Gmail Trigger (`{{ $('Gmail Trigger').item.json.id }}`)  
  - *Input:* From Meetings classification output of Text Classifier.  
  - *Output:* None.  
  - *Version:* 2.1  
  - *Potential Failures:* Label ID invalid; thread ID missing; auth errors.

- **No Operation, do nothing**  
  - *Type:* n8n-nodes-base.noOp  
  - *Role:* Handles emails categorized as Random (or uncategorized) without action.  
  - *Configuration:* No parameters.  
  - *Input:* From Random classification output of Text Classifier.  
  - *Output:* None.  
  - *Version:* 1  
  - *Potential Failures:* None (safe fallback).

---

#### 1.4 Credentials & Documentation

**Overview:**  
Provides a visual sticky note with setup instructions for OpenAI and Gmail credentials to ensure the workflow functions correctly.

**Nodes Involved:**  
- Sticky Note

**Node Details:**

- **Sticky Note**  
  - *Type:* n8n-nodes-base.stickyNote  
  - *Role:* Displays user guidance including links to credential setup.  
  - *Content:*  
    ```
    ## Guide

    Setup your openai credentials: https://platform.openai.com/settings/organization/api-keys

    Setup your gmail credentials
    and you are ready to go: https://console.cloud.google.com/
    ```  
  - *Position:* Top-left corner for visibility.  
  - *Version:* 1  
  - *Potential Failures:* None (informational only).

---

### 3. Summary Table

| Node Name             | Node Type                               | Functional Role                       | Input Node(s)         | Output Node(s)                               | Sticky Note                                                                                     |
|-----------------------|---------------------------------------|-------------------------------------|-----------------------|----------------------------------------------|------------------------------------------------------------------------------------------------|
| Gmail Trigger         | n8n-nodes-base.gmailTrigger            | Input reception of new Gmail emails | None                  | Text Classifier                              | Setup your openai credentials: https://platform.openai.com/settings/organization/api-keys; Setup your gmail credentials and you are ready to go: https://console.cloud.google.com/ |
| Text Classifier       | @n8n/n8n-nodes-langchain.textClassifier | AI-based text classification         | Gmail Trigger         | Business, Cold Emails, Meetings, No Operation |                                                                                                |
| OpenAI Chat Model     | @n8n/n8n-nodes-langchain.lmChatOpenAi | Provides GPT-4 Mini inference        | Text Classifier (ai_languageModel) | Text Classifier                             |                                                                                                |
| Business              | n8n-nodes-base.gmail                   | Adds “IMPORTANT” label to Business emails | Text Classifier (Business) | None                                      |                                                                                                |
| Cold Emails           | n8n-nodes-base.gmail                   | Deletes cold sales email threads     | Text Classifier (Cold Emails) | None                                      |                                                                                                |
| Meetings              | n8n-nodes-base.gmail                   | Adds “YELLOW_STAR” label to meetings | Text Classifier (Meetings) | None                                      |                                                                                                |
| No Operation, do nothing | n8n-nodes-base.noOp                  | Fallback for uncategorized emails    | Text Classifier (Random) | None                                      |                                                                                                |
| Sticky Note           | n8n-nodes-base.stickyNote              | Setup instructions                   | None                  | None                                         | Setup your openai credentials: https://platform.openai.com/settings/organization/api-keys; Setup your gmail credentials and you are ready to go: https://console.cloud.google.com/ |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a “Gmail Trigger” node:**  
   - Type: `n8n-nodes-base.gmailTrigger`  
   - Configure polling to “Every Minute.”  
   - No filters needed (or optionally filter by inbox if desired).  
   - Attach your Gmail OAuth2 credentials (create in n8n if not done).  
   - This node triggers the workflow on new emails.

3. **Add a “Text Classifier” node (LangChain integration):**  
   - Type: `@n8n/n8n-nodes-langchain.textClassifier`  
   - Connect the output of “Gmail Trigger” to this node’s input.  
   - Configure input text as:  
     ```
     subject: {{ $json.Subject }}
     Message: {{ $json.snippet }}
     ```  
   - Define four categories with descriptions:  
     - Business: Anything from friends or co-workers related to work  
     - Sales: Anything sounding like a cold email or sales pitch  
     - Meetings: Anything mentioning meetings, schedules, or reminders  
     - Random: Anything not fitting the above categories  
   - No specific options needed.  
   - This node internally uses a language model for classification.

4. **Add an “OpenAI Chat Model” node:**  
   - Type: `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
   - Set model to `gpt-4.1-mini` (GPT-4 Mini model).  
   - Connect this node’s output to the “Text Classifier” node’s special ai_languageModel input.  
   - Add OpenAI API credentials (create API key on OpenAI platform and configure in n8n).  
   - No extra options needed.

5. **Add Gmail nodes for email handling:**  
   - **Business node:**  
     - Type: `n8n-nodes-base.gmail`  
     - Operation: Add label to thread  
     - Label ID: `IMPORTANT`  
     - Thread ID: `={{ $('Gmail Trigger').item.json.id }}` (dynamic expression)  
     - Credential: Gmail OAuth2  
     - Connect output “Business” from “Text Classifier” to this node.  
  
   - **Cold Emails node:**  
     - Type: `n8n-nodes-base.gmail`  
     - Operation: Delete thread  
     - Thread ID: `={{ $json.id }}` (from classification output)  
     - Credential: Gmail OAuth2  
     - Connect output “Sales” from “Text Classifier” to this node.  
  
   - **Meetings node:**  
     - Type: `n8n-nodes-base.gmail`  
     - Operation: Add label to thread  
     - Label ID: `YELLOW_STAR`  
     - Thread ID: `={{ $('Gmail Trigger').item.json.id }}`  
     - Credential: Gmail OAuth2  
     - Connect output “Meetings” from “Text Classifier” to this node.

6. **Add a “No Operation, do nothing” node:**  
   - Type: `n8n-nodes-base.noOp`  
   - Connect output “Random” from “Text Classifier” to this node.  
   - No configuration needed; acts as a safe fallback.

7. **(Optional) Add a Sticky Note node:**  
   - Type: `n8n-nodes-base.stickyNote`  
   - Content:  
     ```
     ## Guide

     Setup your openai credentials: https://platform.openai.com/settings/organization/api-keys

     Setup your gmail credentials
     and you are ready to go: https://console.cloud.google.com/
     ```  
   - Place prominently for user guidance.

8. **Save and activate the workflow.**

---

### 5. General Notes & Resources

| Note Content                                                                                              | Context or Link                                    |
|-----------------------------------------------------------------------------------------------------------|---------------------------------------------------|
| Setup your OpenAI API keys here: https://platform.openai.com/settings/organization/api-keys              | Required for OpenAI Chat Model node authentication |
| Setup your Gmail OAuth2 credentials here: https://console.cloud.google.com/                               | Required for Gmail Trigger and Gmail action nodes |
| Workflow uses LangChain integration for AI classification; ensure you have the `@n8n/n8n-nodes-langchain` package installed and updated | Node package resource                              |
| Model used is GPT-4 Mini (`gpt-4.1-mini`), a lightweight version of GPT-4, balancing accuracy and cost    | OpenAI model specification                         |

---

**Disclaimer:**  
This document is generated from an automated n8n workflow and complies fully with applicable content policies. All data processed is legal and publicly accessible.