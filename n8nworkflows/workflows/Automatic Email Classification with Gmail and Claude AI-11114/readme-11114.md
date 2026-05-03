Automatic Email Classification with Gmail and Claude AI

https://n8nworkflows.xyz/workflows/automatic-email-classification-with-gmail-and-claude-ai-11114


# Automatic Email Classification with Gmail and Claude AI

### 1. Workflow Overview

This workflow automates the classification and labeling of incoming Gmail messages using AI-powered natural language processing. It targets users who want to organize their inbox automatically by categorizing emails into predefined groups such as Ads, Work, Personal, Financial, and Other. The workflow consists of three major logical blocks:

- **1.1 Gmail Input Reception:** Monitors the Gmail inbox for new incoming emails using a Gmail Trigger node.
- **1.2 AI-Based Email Content Classification:** Utilizes a LangChain-based Email Content Classifier powered by the Anthropic Claude AI model to categorize email content into predefined labels.
- **1.3 Gmail Label Application:** Applies Gmail labels to the email based on the classification result, ensuring automatic inbox organization.

---

### 2. Block-by-Block Analysis

#### 2.1 Gmail Input Reception

**Overview:**  
This block continuously listens for new incoming emails in the connected Gmail account and extracts relevant email content and metadata for further processing.

**Nodes Involved:**  
- Gmail Trigger

**Node Details:**  

- **Gmail Trigger**  
  - **Type & Role:** Gmail Trigger node — monitors Gmail inbox for new emails.  
  - **Configuration:**  
    - Polling mode set to check every minute for new messages.  
    - No specific filtering applied, so it triggers on all new emails.  
    - Connected Gmail OAuth2 credential for authentication.  
  - **Expressions/Variables:** Extracts full email metadata including subject, body, sender, and message IDs.  
  - **Input:** None (trigger node).  
  - **Output:** Passes extracted email content to the Email Content Classifier node.  
  - **Version Requirements:** n8n Gmail Trigger node v1.3 or higher recommended.  
  - **Potential Failures:**  
    - Authentication failure if credentials expire or are revoked.  
    - API rate limits or quota exhaustion could delay triggers.  
    - Polling delay might cause latency in detection.  
  - **Sub-workflow:** None.

#### 2.2 AI-Based Email Content Classification

**Overview:**  
Classifies the content of incoming emails into one of several predefined categories using a natural language processing model, setting the basis for label assignment.

**Nodes Involved:**  
- Anthropic Chat Model  
- Email Content Classifier

**Node Details:**  

- **Anthropic Chat Model**  
  - **Type & Role:** LangChain AI language model node — provides Claude AI model backend.  
  - **Configuration:**  
    - Model selected: "claude-sonnet-4-5-20250929" (Claude Sonnet 4.5 version).  
    - No additional options configured.  
    - Requires Anthropic API credentials.  
  - **Input:** Receives prompt context and text from Email Content Classifier node.  
  - **Output:** Provides AI-generated classification results to Email Content Classifier.  
  - **Version Requirements:** LangChain nodes v1.3+ supporting Anthropic integration.  
  - **Potential Failures:**  
    - API key invalid or quota exceeded.  
    - Timeout or network issues.  
    - Unexpected output format or response errors.  
  - **Sub-workflow:** None.

- **Email Content Classifier**  
  - **Type & Role:** LangChain text classifier node — handles classification logic using the AI model.  
  - **Configuration:**  
    - Input text mapped dynamically from incoming Gmail message content.  
    - Categories defined explicitly:  
      - Advertisment Emails ("Ads")  
      - Work Emails ("Work")  
      - Personal Emails ("Personal")  
      - Financial Emails ("Financial")  
    - Fallback category set to "Other" for unclassified or ambiguous cases.  
    - System prompt instructs AI to output JSON classification without explanations.  
    - Connected to Anthropic Chat Model as the language model backend.  
  - **Input:** Receives email body text from Gmail Trigger.  
  - **Output:** Produces classification label directing flow to one of the Gmail label nodes.  
  - **Version Requirements:** LangChain classifier node v1.1+ supporting AI language model integration.  
  - **Potential Failures:**  
    - Classification errors due to ambiguous email content.  
    - Missing or malformed input text expression.  
    - AI response parsing failure.  
  - **Sub-workflow:** None.

#### 2.3 Gmail Label Application

**Overview:**  
Applies Gmail labels to the email based on the classification result, enabling automatic inbox organization by category.

**Nodes Involved:**  
- Add "Ads" label to message  
- Add "Work" label to message  
- Add "Personal" label to message  
- Add "Financial" label to message  
- Add "Other" label to message

**Node Details:**  

Each node shares similar configuration principles, differing only by the label applied:

- **Node Type & Role:** Gmail node — performs label addition operation on the email identified by message ID.  
- **Configuration:**  
  - Operation: `addLabels`  
  - Label to add corresponds to the node name, e.g., "Ads", "Work", "Personal", "Financial", or "Other".  
  - Uses the same Gmail OAuth2 credential as the trigger node.  
  - Receives classification result routing to select the correct label application node.  
- **Input:** Receives email metadata including message ID from Email Content Classifier output.  
- **Output:** Confirms success of label addition.  
- **Version Requirements:** Gmail node v2.1+ recommended for label operations.  
- **Potential Failures:**  
  - Authentication issues with Gmail OAuth2 credential.  
  - Label not existing in the Gmail account or misspelled label names.  
  - API quota exceeded or Gmail API errors.  
  - Message ID invalid or message deleted before label application.  
- **Sub-workflow:** None.

---

### 3. Summary Table

| Node Name                     | Node Type                               | Functional Role                           | Input Node(s)           | Output Node(s)                                                                                  | Sticky Note                                                                                                  |
|-------------------------------|---------------------------------------|-----------------------------------------|------------------------|------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------|
| Gmail Trigger                 | Gmail Trigger                         | Incoming email listener                  | None                   | Email Content Classifier                                                                        | ## Trigger  Gmail Trigger — listens for new incoming messages.                                               |
| Email Content Classifier      | LangChain Text Classifier             | Classify email content into categories  | Gmail Trigger          | Add "Ads" label to message, Add "Work" label to message, Add "Personal" label to message, Add "Financial" label to message, Add "Other" label to message | ## Classification  Email Content Classifier (LangChain) — classifies message content into category labels.  |
| Anthropic Chat Model          | LangChain AI Language Model (Anthropic) | AI model backend for classification     | Email Content Classifier (ai_languageModel input) | Email Content Classifier                                                                        | ## Model  Anthropic Chat Model — language model used by classifier (model selection / credential).            |
| Add "Ads" label to message   | Gmail                                | Apply "Ads" label                       | Email Content Classifier | None                                                                                           | ## Label Actions  Gmail nodes — Add "Ads", "Work", "Personal", "Financial", or "Other" labels.                |
| Add "Work" label to message  | Gmail                                | Apply "Work" label                      | Email Content Classifier | None                                                                                           | ## Label Actions  Gmail nodes — Add "Ads", "Work", "Personal", "Financial", or "Other" labels.                |
| Add "Personal" label to message | Gmail                              | Apply "Personal" label                  | Email Content Classifier | None                                                                                           | ## Label Actions  Gmail nodes — Add "Ads", "Work", "Personal", "Financial", or "Other" labels.                |
| Add "Financial" label to message | Gmail                             | Apply "Financial" label                 | Email Content Classifier | None                                                                                           | ## Label Actions  Gmail nodes — Add "Ads", "Work", "Personal", "Financial", or "Other" labels.                |
| Add "Other" label to message | Gmail                                | Apply "Other" fallback label            | Email Content Classifier | None                                                                                           | ## Label Actions  Gmail nodes — Add "Ads", "Work", "Personal", "Financial", or "Other" labels.                |
| Sticky Note                   | Sticky Note                          | Workflow explanation and usage notes    | None                   | None                                                                                           | ## How It Works (Detailed workflow explanation and setup instructions)                                       |
| Sticky Note1                  | Sticky Note                          | Classification block description        | None                   | None                                                                                           | ## Classification  Email Content Classifier (LangChain) — classifies message content into category labels.  |
| Sticky Note2                  | Sticky Note                          | AI Model block description               | None                   | None                                                                                           | ## Model  Anthropic Chat Model — language model used by classifier (model selection / credential).            |
| Sticky Note3                  | Sticky Note                          | Gmail label actions description          | None                   | None                                                                                           | ## Label Actions  Gmail nodes — Add "Ads", "Work", "Personal", "Financial", or "Other" labels.                |
| Sticky Note4                  | Sticky Note                          | Gmail trigger block description          | None                   | None                                                                                           | ## Trigger  Gmail Trigger — listens for new incoming messages.                                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Gmail Trigger Node:**  
   - Type: Gmail Trigger  
   - Configure with your Gmail OAuth2 credentials.  
   - Set polling to “Every Minute” for checking new emails.  
   - No filters needed unless you want to restrict monitored emails.

2. **Create Anthropic Chat Model Node:**  
   - Type: LangChain AI Language Model (Anthropic)  
   - Configure with Anthropic API credentials.  
   - Select model: "claude-sonnet-4-5-20250929" (Claude Sonnet 4.5).  
   - Leave options default.

3. **Create Email Content Classifier Node:**  
   - Type: LangChain Text Classifier  
   - Connect AI language model input to the Anthropic Chat Model node.  
   - Set input text to dynamically map from Gmail Trigger message content (e.g., email body).  
   - Define categories exactly as:  
     - Advertisment Emails (label: "Ads")  
     - Work Emails (label: "Work")  
     - Personal Emails (label: "Personal")  
     - Financial Emails (label: "Financial")  
   - Set fallback category to “Other.”  
   - Use system prompt instructing AI to output JSON classification only, no explanations.

4. **Connect Gmail Trigger output to Email Content Classifier input.**

5. **Create Gmail Nodes for Label Application:**  
   For each label ("Ads", "Work", "Personal", "Financial", "Other"):  
   - Type: Gmail  
   - Operation: addLabels  
   - Connect Gmail OAuth2 credential.  
   - Configure label name to match exactly the Gmail label in your account (e.g., "Ads").  
   - Connect corresponding output from Email Content Classifier to each Gmail label node respectively (classification result determines routing).

6. **Connect Email Content Classifier outputs to each Gmail label node:**  
   - The classifier node provides multiple outputs (one per category), link each to the respective label node.

7. **Activate the workflow and test:**  
   - Send test emails to your Gmail account with varied content.  
   - Confirm labels are applied automatically.  
   - Adjust category names in classifier or Gmail labels if mismatches occur.

---

### 5. General Notes & Resources

| Note Content                                                                                           | Context or Link                                                                                              |
|------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------|
| Detailed explanation of workflow logic, setup steps, use cases, customization options, and troubleshooting tips are embedded as a sticky note within the workflow. | See Sticky Note at position [-1152,-480] named "Sticky Note".                                                |
| Gmail labels must exist in your Gmail account before running the workflow to ensure correct application. | Gmail Label Management — ensure labels Ads, Work, Personal, Financial, Other are predefined.                 |
| Anthropic Claude AI model requires valid API credentials and may incur usage costs; monitor usage accordingly. | Anthropic API documentation: https://www.anthropic.com/index/api                                              |
| The workflow is optimized for n8n versions supporting LangChain nodes and Gmail OAuth2 integrations.  | Recommended n8n version 0.205+ for LangChain and Gmail node updates.                                         |

---

_Disclaimer: The provided text is extracted exclusively from an automated n8n workflow. This processing fully complies with current content policies and contains no illegal, offensive, or copyrighted material. All data handled is legal and publicly accessible._