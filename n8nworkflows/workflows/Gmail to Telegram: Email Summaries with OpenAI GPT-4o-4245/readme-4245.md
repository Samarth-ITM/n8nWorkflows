Gmail to Telegram: Email Summaries with OpenAI GPT-4o

https://n8nworkflows.xyz/workflows/gmail-to-telegram--email-summaries-with-openai-gpt-4o-4245


# Gmail to Telegram: Email Summaries with OpenAI GPT-4o

### 1. Workflow Overview

This workflow automates the process of summarizing incoming Gmail emails using OpenAI’s GPT-4o model and sending these AI-generated summaries to a specified Telegram chat. It is designed for users who want quick, concise, and informal email summaries delivered directly to Telegram for ease of access and timely updates.

The workflow is logically structured into these main blocks:

- **1.1 Input Reception:** Detects new incoming emails from a Gmail account.
- **1.2 Summary Language Setup:** Sets the preferred language for the summary.
- **1.3 Data Preparation:** Extracts and prepares relevant email fields for summarization.
- **1.4 AI Summary Generation:** Uses OpenAI GPT-4o via Langchain to generate a natural, informal summary of the email.
- **1.5 Output Delivery:** Sends the generated summary to a Telegram chat.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block triggers the workflow whenever a new email arrives in the connected Gmail inbox.

- **Nodes Involved:**  
  - When a new email arrives

- **Node Details:**  

  - **When a new email arrives**  
    - *Type & Role:* Gmail Trigger node; initiates workflow on new email reception.  
    - *Configuration:* Polls Gmail every minute for new emails; does not filter by labels or senders, so all incoming emails trigger the workflow.  
    - *Key Expressions:* None.  
    - *Connections:* Outputs data to "Set summary language" node.  
    - *Version Requirements:* Version 1.2; requires valid Gmail OAuth2 credentials.  
    - *Failure Modes:* Authentication errors (expired or invalid Gmail OAuth2 token), Gmail API rate limits, network timeouts.

#### 2.2 Summary Language Setup

- **Overview:**  
  Sets the language in which the email summary will be generated. Defaults to English, but can be customized.

- **Nodes Involved:**  
  - Set summary language

- **Node Details:**  

  - **Set summary language**  
    - *Type & Role:* Set node; assigns a static value to the field `summary_language`.  
    - *Configuration:* Sets `summary_language` to `"english"`. Passes through all other incoming fields unchanged.  
    - *Key Expressions:* None dynamic; hardcoded language value.  
    - *Connections:* Outputs to "Prepare fields for agent".  
    - *Version Requirements:* Version 3.4.  
    - *Failure Modes:* None expected; static assignment.

#### 2.3 Data Preparation

- **Overview:**  
  Extracts critical email information (sender, subject, message body) and ensures these fields are correctly formatted for the AI summarization agent.

- **Nodes Involved:**  
  - Prepare fields for agent

- **Node Details:**  

  - **Prepare fields for agent**  
    - *Type & Role:* Set node; extracts and normalizes email fields for use in prompts.  
    - *Configuration:*  
      - `summary_language` is passed through from input JSON.  
      - `from`: extracts sender email or text from multiple possible JSON paths; defaults to `"Unknown sender"` if missing.  
      - `subject`: extracts email subject.  
      - `message`: prefers `html` content, then `text`, or defaults to `"No message content available"`.  
    - *Key Expressions:*  
      - `={{ $json.from?.text || $json.from?.value?.[0]?.address || 'Unknown sender' }}`  
      - `={{ $json.html || $json.text || 'No message content available' }}`  
    - *Connections:* Outputs to "Summary generation agent".  
    - *Version Requirements:* Version 3.4.  
    - *Failure Modes:* Missing or malformed email fields may cause incomplete summaries; expression evaluation errors if unexpected JSON structure.

#### 2.4 AI Summary Generation

- **Overview:**  
  This block generates a concise, informal summary of the email content using a Langchain agent with OpenAI's GPT-4o model.

- **Nodes Involved:**  
  - OpenAI Model  
  - Summary generation agent

- **Node Details:**  

  - **OpenAI Model**  
    - *Type & Role:* Language model node; provides GPT-4o-mini model access for Langchain.  
    - *Configuration:* Uses model `gpt-4o-mini` with no additional options.  
    - *Credentials:* OpenAI API key required.  
    - *Connections:* Outputs to "Summary generation agent" as the language model.  
    - *Version Requirements:* Version 1.2; requires valid OpenAI API credentials.  
    - *Failure Modes:* API key errors, rate limits, network timeouts.

  - **Summary generation agent**  
    - *Type & Role:* Langchain Agent node; orchestrates prompt construction and calls the language model to generate the summary.  
    - *Configuration:*  
      - Prompt text dynamically constructed with fields:  
        ```
        Summarize the following email in this language: {{ $json.summary_language }}

        From: {{ $json.from }}
        Subject: {{ $json.subject }}

        {{ $json.message }}
        ```  
      - System message instructs a relaxed, natural, informal tone, focusing on relevant details and avoiding robotic or redundant language.  
      - Retries on failure with a 3-second wait between tries.  
    - *Connections:* Outputs summary to "Send summary to Telegram".  
    - *Version Requirements:* Version 1.9.  
    - *Failure Modes:* API errors, expression resolution failures, prompt misformatting, rate limits.

#### 2.5 Output Delivery

- **Overview:**  
  Sends the AI-generated summary text to a Telegram chat via a bot.

- **Nodes Involved:**  
  - Send summary to Telegram

- **Node Details:**  

  - **Send summary to Telegram**  
    - *Type & Role:* Telegram node; sends text messages to a specified chat.  
    - *Configuration:*  
      - Sends message text from `{{ $json.output }}` (summary generated by agent).  
      - Chat ID must be set by the user (placeholder text present).  
      - Uses Markdown parse mode to format messages.  
      - Does not append attribution text.  
      - Retries on fail with 3-second intervals.  
    - *Credentials:* Telegram API credentials (bot token).  
    - *Connections:* Terminal node; no outputs.  
    - *Version Requirements:* Version 1.2.  
    - *Failure Modes:* Invalid bot token, incorrect chat ID, Telegram API downtime.

---

### 3. Summary Table

| Node Name               | Node Type                              | Functional Role                         | Input Node(s)             | Output Node(s)              | Sticky Note                                                                                                                        |
|-------------------------|--------------------------------------|---------------------------------------|---------------------------|-----------------------------|-----------------------------------------------------------------------------------------------------------------------------------|
| When a new email arrives | Gmail Trigger                        | Trigger workflow on new Gmail email   | —                         | Set summary language         | ## 📨 Send AI summaries of incoming emails from Gmail to Telegram<br><br>### ℹ️ ABOUT THIS WORKFLOW<br>This workflow will send you AI-generated summaries of all your Gmail incoming emails to Telegram.<br><br>### 🛠️ SETUP<br>1. Create credentials for Gmail, Telegram, and OpenAI.<br>2. Add the Gmail credential to the Gmail node.<br>3. Add the OpenAI credential to the OpenAI Model node.<br>4. Add the Telegram credential to the Telegram node.<br>5. Set your Chat ID in the Telegram node; you can find it out by using Telegram's get ID bot.<br><br>### 🎨 CUSTOMIZING THE SUMMARIES<br>You can customize the generated summaries by:<br>- Changing the language in the "Set summary language" node. You can type anything you want here, you can even specify a specific zone dialect.<br>- Modifying the prompts (in the "Summary generation agent" node)<br>- Selecting another model (in the "OpenAI Model" node)<br>- You can also replace the "OpenAI Model" node with another provider, such as Anthropic or DeepSeek. |
| Set summary language     | Set                                  | Set the language for the summary      | When a new email arrives   | Prepare fields for agent     | See above sticky note                                                                                                              |
| Prepare fields for agent | Set                                  | Normalize and prepare email fields    | Set summary language       | Summary generation agent     | See above sticky note                                                                                                              |
| OpenAI Model             | Langchain LM Chat OpenAI             | Provide GPT-4o language model         | —                         | Summary generation agent (lm) | See above sticky note                                                                                                              |
| Summary generation agent | Langchain Agent                      | Generate summary text from email data | Prepare fields for agent, OpenAI Model | Send summary to Telegram | See above sticky note                                                                                                              |
| Send summary to Telegram | Telegram                            | Send summary text message to Telegram | Summary generation agent   | —                           | See above sticky note                                                                                                              |
| Sticky Note              | Sticky Note                         | Informational setup and customization | —                         | —                           | See content in “Sticky Note” node above                                                                                           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Gmail Trigger node**  
   - Type: Gmail Trigger  
   - Credentials: Connect Gmail OAuth2 credential for your email account.  
   - Settings: Poll every minute without filters.  
   - Name it: "When a new email arrives".

2. **Create Set node to define summary language**  
   - Type: Set  
   - Add a field: `summary_language` (string) with value `"english"` (or your preferred language).  
   - Enable "Keep Only Set" = false (pass other fields).  
   - Name it: "Set summary language".  
   - Connect "When a new email arrives" → "Set summary language".

3. **Create Set node to prepare email fields**  
   - Type: Set  
   - Add fields with expressions:  
     - `summary_language`: `={{ $json.summary_language }}`  
     - `from`: `={{ $json.from?.text || $json.from?.value?.[0]?.address || 'Unknown sender' }}`  
     - `subject`: `={{ $json.subject }}`  
     - `message`: `={{ $json.html || $json.text || 'No message content available' }}`  
   - Name it: "Prepare fields for agent".  
   - Connect "Set summary language" → "Prepare fields for agent".

4. **Create Langchain LM Chat OpenAI node**  
   - Type: Langchain LM Chat OpenAI  
   - Credentials: Connect your OpenAI API key credential.  
   - Model: Select `gpt-4o-mini`.  
   - Name it: "OpenAI Model".

5. **Create Langchain Agent node**  
   - Type: Langchain Agent  
   - Parameters:  
     - Text prompt:  
       ```
       Summarize the following email in this language: {{ $json.summary_language }}

       From: {{ $json.from }}
       Subject: {{ $json.subject }}

       {{ $json.message }}
       ```  
     - System message:  
       ```
       You summarize emails in a short, natural, and informal way. Use a casual tone, like you're talking to a friend. Always write in the language specified by the user. Include who sent the email, what it’s about, the most relevant details (like purchase info, prices, discounts, dates, or refund conditions), and ignore anything redundant or overly formal. Avoid robotic language, lists, or instructions like “keep this email.” Just say what matters.
       ```  
     - Retry on fail: Enable with 3 seconds wait.  
   - Name it: "Summary generation agent".  
   - Connect "Prepare fields for agent" → "Summary generation agent" (main).  
   - Connect "OpenAI Model" → "Summary generation agent" (ai_languageModel input).

6. **Create Telegram node**  
   - Type: Telegram  
   - Credentials: Connect your Telegram bot API token credential.  
   - Parameters:  
     - Text: `={{ $json.output }}` (the generated summary)  
     - Chat ID: Set your Telegram chat ID here (replace placeholder).  
     - Additional fields: Set parse mode to Markdown, disable appending attribution.  
   - Retry on fail: Enable, with 3 seconds wait.  
   - Name it: "Send summary to Telegram".  
   - Connect "Summary generation agent" → "Send summary to Telegram".

7. **Optional: Add a Sticky Note node**  
   - To provide instructions or documentation inside the workflow canvas, create a sticky note node with setup instructions and usage notes.

8. **Activate the workflow**  
   - Ensure all credentials are valid and tested.  
   - Turn on the workflow to start receiving email summaries on Telegram.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                    | Context or Link                                                                                                                                                     |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| This workflow sends AI-generated summaries of all incoming Gmail emails to Telegram. Customization is available by changing the language, the prompt, or the AI model used. You can replace the OpenAI Model node with other providers like Anthropic or DeepSeek. The Telegram Chat ID can be found using Telegram's "get ID" bot.                                                                                                                                                           | Sticky Note inside the workflow; useful for onboarding and customization guidance.                                                                                 |
| The summary tone is informal and casual, targeting natural language suitable for quick reading, avoiding formal or robotic styles. This is controlled by the system message of the Langchain agent.                                                                                                                                                                                                                                                                                  | Embedded in the "Summary generation agent" node configuration.                                                                                                    |
| For Telegram Bot setup and obtaining Chat IDs, consult Telegram’s official bot documentation and common tools like "@userinfobot" or similar to get chat IDs.                                                                                                                                                                                                                                                                                                                       | Telegram Bot API documentation and community tools.                                                                                                              |
| Gmail OAuth2 credentials require enabling Gmail API and configuring consent screen in Google Cloud Console. Ensure scopes include reading emails.                                                                                                                                                                                                                                                                                                                                  | Google Cloud Platform and Gmail API setup documentation.                                                                                                         |
| To avoid API rate limits, consider adding error handling and exponential backoff or splitting the workflow for heavy email volumes.                                                                                                                                                                                                                                                                                                                                              | Best practices for API usage in n8n and OpenAI.                                                                                                                  |

---

**Disclaimer:**  
The text provided comes exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.