Classify Gmail Emails with GPT-4o mini and Send Telegram Notifications

https://n8nworkflows.xyz/workflows/classify-gmail-emails-with-gpt-4o-mini-and-send-telegram-notifications-7553


# Classify Gmail Emails with GPT-4o mini and Send Telegram Notifications

### 1. Workflow Overview

This workflow automates the classification of incoming Gmail emails using GPT-4o mini AI models and sends Telegram notifications about each email. It is designed for users who want to automatically organize and prioritize their email inbox with AI-powered labeling and receive real-time notifications on Telegram.

**Target Use Cases:**  
- Automatic categorization of emails into "High Priority," "Work Related," "Promotion," or "Other."  
- Adding Gmail labels to emails based on AI classification to improve inbox management.  
- Sending concise Telegram notifications for incoming emails to stay updated without checking the inbox constantly.  

**Logical Blocks:**  
- **1.1 Input Reception:** Listening for new Gmail emails in real time.  
- **1.2 AI Classification:** Using LangChain text classification with GPT-4o mini to categorize the email content.  
- **1.3 Gmail Labeling:** Applying Gmail labels based on classification results.  
- **1.4 Notification Generation:** Creating short, direct Telegram messages describing the email.  
- **1.5 Telegram Notification:** Sending the generated notification to a Telegram chat.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
This block triggers the workflow every minute when new Gmail emails arrive, providing the raw email data for processing.

**Nodes Involved:**  
- Gmail Trigger  
- Sticky Note3 (documentation)

**Node Details:**  

- **Gmail Trigger**  
  - Type: Gmail Trigger node (n8n-nodes-base.gmailTrigger)  
  - Role: Watches Gmail inbox for new emails, polling every minute.  
  - Configuration: Polling mode set to “everyMinute,” no filters applied (listens to all incoming emails).  
  - Credentials: Uses OAuth2 Gmail credentials configured for the user’s Gmail account.  
  - Inputs: None (trigger node).  
  - Outputs: Emits JSON data containing full email details upon new email arrival.  
  - Edge cases: Gmail API limits or OAuth token expiry could cause failures; network issues may delay or miss events.  
- **Sticky Note3**  
  - Role: Provides user instructions about Gmail Trigger node polling interval.  
  - Content: "# Mail Listener\nThis node listens for new emails every minute, you may change the polling time"  
  - No technical behavior.

---

#### 2.2 AI Classification

**Overview:**  
This block uses a LangChain text classification node powered by GPT-4o mini to assign one of several predefined categories to the incoming email content.

**Nodes Involved:**  
- Classification Agent  
- Sticky Note2 (documentation)

**Node Details:**  

- **Classification Agent**  
  - Type: LangChain Text Classifier node (@n8n/n8n-nodes-langchain.textClassifier)  
  - Role: Classifies email content into categories: High Priority, Work Related, Promotion, or Other.  
  - Configuration:  
    - Multi-class classification enabled.  
    - System prompt instructs the model to classify user-provided text strictly into specified categories as JSON output, no explanation.  
    - Input text template includes email sender address, sender name, subject, and body text from Gmail Trigger data.  
    - Categories are detailed with descriptions and keywords for clarity to the model.  
  - Inputs: Receives email JSON from Gmail Trigger.  
  - Outputs: Classification result with label indicating email category.  
  - Edge cases: Misclassification possible if email content is ambiguous; model latency or API errors; invalid email fields could cause expression failures.  
- **Sticky Note2**  
  - Role: Documents that categories can be customized according to user needs.  
  - Content: "## AI Classification\nYou may Customize the Categories based on the Emails you want to be labeled."

---

#### 2.3 Gmail Labeling

**Overview:**  
Based on classification results, this block applies the corresponding Gmail labels to the email to help organize the inbox automatically.

**Nodes Involved:**  
- High Priority  
- Work Related  
- Promotions  
- Sticky Note (documentation)

**Node Details:**  

- **High Priority**  
  - Type: Gmail node (n8n-nodes-base.gmail)  
  - Role: Adds "YELLOW_STAR" and "IMPORTANT" labels to emails classified as High Priority.  
  - Configuration:  
    - Operation: addLabels.  
    - LabelIds: ["YELLOW_STAR", "IMPORTANT"].  
    - MessageId: Extracted dynamically from email data.  
  - Credentials: Uses the same Gmail OAuth2 credentials as the trigger.  
  - Inputs: Email JSON with classification result.  
  - Outputs: Confirmation of label addition.  
  - Edge cases: Labels must exist in Gmail account; missing labels cause failure; API rate limits or invalid message IDs cause errors.  

- **Work Related**  
  - Type: Gmail node (n8n-nodes-base.gmail)  
  - Role: Adds a custom label "Label_1704671005251458060" to emails classified as Work Related.  
  - Configuration: Similar to High Priority node but with different label ID.  
  - Edge cases: Same as High Priority node.  

- **Promotions**  
  - Type: Gmail node (n8n-nodes-base.gmail)  
  - Role: Adds a custom label "Label_2537748930215029681" to emails classified as Promotions.  
  - Configuration: Similar to above nodes.  
  - Edge cases: Same as above.  

- **Sticky Note**  
  - Role: Explains Gmail labeling and important note on label existence.  
  - Content:  
    "## Gmail Label\nThis node is responsible for actually labeling your email.\nYou may add 2 labels in one classification but for this template, I did a 1 Classification, 1 Label\n\n## NOTE:\nMake sure the label name you'll add in your label node is already existing in your gmail account otherwise it won't work."

---

#### 2.4 Notification Generation

**Overview:**  
Generates a short, casual Telegram notification message describing the email, including the classified category, sender, subject, and body snippet.

**Nodes Involved:**  
- AI Agent1  
- 4o-mini  
- 4o- mini  
- Sticky Note1 (documentation)

**Node Details:**  

- **AI Agent1**  
  - Type: LangChain Agent node (@n8n/n8n-nodes-langchain.agent)  
  - Role: Generates notification text for Telegram based on email and classification data.  
  - Configuration:  
    - Prompt instructs the agent to create a short, casual, direct notification with category, sender (address or name), subject, and body.  
    - Uses expressions to inject these fields dynamically from the email JSON.  
  - Inputs: Classification results and email JSON.  
  - Outputs: Generated notification text.  
  - Edge cases: If classification label missing or JSON fields empty, notification may be incomplete; API errors or timeouts possible.  
- **4o-mini** and **4o- mini**  
  - Type: OpenAI GPT-4o mini language model nodes (@n8n/n8n-nodes-langchain.lmChatOpenAi)  
  - Role: Provide GPT-4o mini model integration for the LangChain nodes (Classification Agent and AI Agent1).  
  - Configuration: Model set to "gpt-4o-mini," no special options.  
  - Credentials: Uses OpenAI API credentials.  
  - Inputs: Text prompts from LangChain nodes, outputs generated text responses.  
  - Edge cases: API quota or authentication issues; model latency; version compatibility with n8n LangChain nodes.  

- **Sticky Note1**  
  - Role: Provides instructions for Telegram bot setup and chat ID configuration.  
  - Content:  
    "## Telegram Message\n- Create a bot on @botfather telegram and use that as a credential.\n- Set the chat ID as your chatID so it would message you."

---

#### 2.5 Telegram Notification

**Overview:**  
Sends the generated notification message to a specified Telegram chat.

**Nodes Involved:**  
- Send a text message

**Node Details:**  

- **Send a text message**  
  - Type: Telegram node (n8n-nodes-base.telegram)  
  - Role: Sends text messages to a Telegram user or group using a bot.  
  - Configuration:  
    - Text: Uses the output text generated by AI Agent1.  
    - Chat ID: User must enter their Telegram chat ID manually (placeholder in workflow).  
  - Credentials: Uses Telegram API credentials tied to a bot created via BotFather.  
  - Inputs: Notification text from AI Agent1.  
  - Outputs: Confirmation of message sent.  
  - Edge cases: Invalid chat ID or revoked bot token will cause failure; network or API errors possible.  

---

### 3. Summary Table

| Node Name           | Node Type                                   | Functional Role                      | Input Node(s)           | Output Node(s)                      | Sticky Note                                                                                                       |
|---------------------|---------------------------------------------|------------------------------------|------------------------|-----------------------------------|-------------------------------------------------------------------------------------------------------------------|
| Gmail Trigger       | n8n-nodes-base.gmailTrigger                  | Listens for new Gmail emails       | None                   | Classification Agent              | # Mail Listener\nThis node listens for new emails every minute, you may change the polling time                   |
| Classification Agent | @n8n/n8n-nodes-langchain.textClassifier     | Classifies email content            | Gmail Trigger          | High Priority, Work Related, Promotions, AI Agent1 | ## AI Classification\nYou may Customize the Categories based on the Emails you want to be labeled.               |
| High Priority       | n8n-nodes-base.gmail                         | Adds High Priority Gmail labels     | Classification Agent   | None                             | ## Gmail Label\nThis node is responsible for actually labeling your email.\nMake sure the label exists in Gmail.  |
| Work Related        | n8n-nodes-base.gmail                         | Adds Work Related Gmail labels      | Classification Agent   | None                             | ## Gmail Label\nThis node is responsible for actually labeling your email.\nMake sure the label exists in Gmail.  |
| Promotions          | n8n-nodes-base.gmail                         | Adds Promotions Gmail labels        | Classification Agent   | None                             | ## Gmail Label\nThis node is responsible for actually labeling your email.\nMake sure the label exists in Gmail.  |
| AI Agent1           | @n8n/n8n-nodes-langchain.agent              | Generates Telegram notification text| Classification Agent   | Send a text message              | ## Telegram Message\n- Create a bot on @botfather telegram and use that as a credential.\n- Set your chat ID.     |
| Send a text message | n8n-nodes-base.telegram                      | Sends notification to Telegram      | AI Agent1              | None                             | ## Telegram Message\n- Create a bot on @botfather telegram and use that as a credential.\n- Set your chat ID.     |
| 4o-mini             | @n8n/n8n-nodes-langchain.lmChatOpenAi       | GPT-4o mini model for AI Agent1     | AI Agent1              | AI Agent1 (response)             |                                                                                                                   |
| 4o- mini            | @n8n/n8n-nodes-langchain.lmChatOpenAi       | GPT-4o mini model for Classification | Classification Agent   | Classification Agent (response)  |                                                                                                                   |
| Sticky Note         | n8n-nodes-base.stickyNote                    | Documentation on Gmail Labeling     | None                   | None                            | ## Gmail Label\nThis node is responsible for labeling your email.\nLabels must exist in Gmail.                    |
| Sticky Note1        | n8n-nodes-base.stickyNote                    | Documentation on Telegram setup     | None                   | None                            | ## Telegram Message\n- Create a bot on @botfather telegram and set chat ID.                                       |
| Sticky Note2        | n8n-nodes-base.stickyNote                    | Documentation on AI Classification  | None                   | None                            | ## AI Classification\nCustomize categories as needed.                                                            |
| Sticky Note3        | n8n-nodes-base.stickyNote                    | Documentation on Gmail Trigger      | None                   | None                            | # Mail Listener\nThis node listens for new emails every minute, configurable polling time.                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Gmail Trigger node:**  
   - Type: Gmail Trigger  
   - Set to poll every minute (or desired interval).  
   - Use OAuth2 credentials connected to your Gmail account.  
   - No filters (to listen to all incoming emails).  

2. **Create Classification Agent node (LangChain Text Classifier):**  
   - Type: @n8n/n8n-nodes-langchain.textClassifier  
   - Configure multiClass classification.  
   - System prompt: instruct model to classify email text into categories (High Priority, Work Related, Promotion, Other) and output JSON only.  
   - Input text using expression:  
     ```
     Email: {{ $json.from.value[0].address }}
     Name: {{ $json.from.value[0].name }}
     Subject: {{ $json.subject }}
     Body: {{ $json.text }}
     ```  
   - Define categories with descriptions and keywords accordingly.  
   - Connect OpenAI GPT-4o mini credentials to an LM Chat node (see step 3).  

3. **Create GPT-4o mini LM Chat nodes (two instances):**  
   - Type: @n8n/n8n-nodes-langchain.lmChatOpenAi  
   - Model: gpt-4o-mini  
   - Credentials: Configure OpenAI account API key.  
   - One LM node connects as language model for Classification Agent.  
   - Second LM node connects as language model for AI Agent1 (next step).  

4. **Connect Gmail Trigger to Classification Agent node.**

5. **Create Gmail nodes for labeling:**  
   - For "High Priority":  
     - Type: Gmail node  
     - Operation: addLabels  
     - Label IDs: "YELLOW_STAR" and "IMPORTANT" (ensure these labels exist in Gmail)  
     - Use messageId from email JSON (`={{ $json.id }}`)  
     - Use same Gmail OAuth2 credentials.  
   - For "Work Related":  
     - Similar setup, with your custom label ID (e.g., "Label_1704671005251458060").  
   - For "Promotion":  
     - Similar setup, with your custom label ID (e.g., "Label_2537748930215029681").  

6. **Connect Classification Agent outputs to each Gmail labeling node using separate branches:**  
   - Branch 1: High Priority  
   - Branch 2: Work Related  
   - Branch 3: Promotions  

7. **Create AI Agent1 node (LangChain Agent):**  
   - Type: @n8n/n8n-nodes-langchain.agent  
   - Prompt:  
     ```
     You are a notification assistant. When an email is received, generate a short, casual, and direct notification about it.
     CATEGORY:  {{ $json.labelIds[0] }}
     From: {{ $json.from.value[0].address  || $json.from.value[0].name}} 

     Subject: {{ $json.subject }}
     Body: {{ $json.text }}
     ```  
   - Connect OpenAI GPT-4o mini credentials to LM Chat node created in step 3.  
   - Connect Classification Agent node outputs also to AI Agent1 (all branches).  

8. **Create Telegram node to send message:**  
   - Type: Telegram  
   - Text: Use expression `={{ $json.output }}` (output from AI Agent1)  
   - Chat ID: Enter your Telegram chat ID (numeric or username as configured)  
   - Credentials: Use Telegram API credentials created from a bot via BotFather.  

9. **Connect AI Agent1 output to Telegram node input.**

10. **Optional:** Add Sticky Note nodes for documentation within your workflow to explain each block and key configuration points.

---

### 5. General Notes & Resources

| Note Content                                                                                                         | Context or Link                                      |
|----------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------|
| Create a Telegram bot via @BotFather and use its token for Telegram API credentials.                                 | Telegram bot creation instructions (@BotFather)    |
| Ensure all Gmail labels used exist in your Gmail account to avoid label application failures.                        | Gmail label management documentation                 |
| Customize AI classification categories and prompt templates to better fit your email management needs.              | LangChain text classification best practices        |
| Telegram chat ID must be correctly set to receive messages; use a bot to send messages to yourself or groups.       | Telegram API chat ID retrieval tips                   |
| GPT-4o mini model requires valid OpenAI API credentials and may have usage quotas or costs.                          | OpenAI GPT-4o-mini documentation                      |

---

**Disclaimer:**  
The text provided is exclusively derived from an n8n automated workflow. The processing strictly complies with applicable content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.