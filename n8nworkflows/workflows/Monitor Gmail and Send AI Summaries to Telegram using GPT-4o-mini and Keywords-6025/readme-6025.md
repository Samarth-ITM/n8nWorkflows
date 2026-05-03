Monitor Gmail and Send AI Summaries to Telegram using GPT-4o-mini and Keywords

https://n8nworkflows.xyz/workflows/monitor-gmail-and-send-ai-summaries-to-telegram-using-gpt-4o-mini-and-keywords-6025


# Monitor Gmail and Send AI Summaries to Telegram using GPT-4o-mini and Keywords

---

### 1. Workflow Overview

This workflow automates monitoring a Gmail inbox for new emails, filters them based on importance using keyword matching, summarizes the important emails using OpenAI’s GPT-4o-mini model, and sends concise, human-friendly summaries to a Telegram chat. It is designed for users who want quick, Telegram-style notifications about critical emails such as billing alerts, delivery updates, job offers, or security messages.

The workflow is logically divided into the following blocks:

- **1.1 Email Reception:** Triggered every minute by new Gmail messages.
- **1.2 Importance Filtering:** Checks if the email subject or body contains predefined keywords.
- **1.3 AI Summarization:** Sends filtered email content to GPT-4o-mini for a short Telegram-style summary.
- **1.4 Telegram Notification:** Sends the generated summary text to a specified Telegram chat.

---

### 2. Block-by-Block Analysis

#### 2.1 Email Reception

- **Overview:** Polls the Gmail inbox every minute for new emails, capturing the subject and body text.
- **Nodes Involved:**  
  - Check for New Emails (Gmail Trigger)

- **Node Details:**

  - **Check for New Emails**  
    - *Type:* Gmail Trigger  
    - *Role:* Entry point to detect new incoming emails.  
    - *Configuration:*  
      - Polls every minute (set in `pollTimes` with mode `everyMinute`).  
      - Retrieves full email content (`simple` set to false) to access subject and body text.  
      - No filters applied at this stage; all new emails trigger the workflow.  
    - *Inputs:* None (trigger node).  
    - *Outputs:* Emits new email data for downstream processing.  
    - *Potential failure modes:* Authentication errors (OAuth2 token expiration), Gmail API rate limits or downtime, network issues causing polling failures.

#### 2.2 Importance Filtering

- **Overview:** Determines if an email is important by checking if the subject or email body contains any of a configured set of keywords. Only important emails proceed to summarization.
- **Nodes Involved:**  
  - Important Email Filter (IF node)  
  - Ignore Unimportant Email (NoOp node)

- **Node Details:**

  - **Important Email Filter**  
    - *Type:* IF (conditional) node  
    - *Role:* Evaluates if the email matches importance criteria.  
    - *Configuration:*  
      - Uses an OR combinator with two conditions:  
        1. Subject contains `"sales"` (case sensitive).  
        2. Email body text contains `"jobs"` (case sensitive).  
      - These keywords can be customized to match other important terms like `"invoice"`, `"payment due"`, `"security alert"`, etc.  
    - *Expressions:*  
      - `{{$json.subject}}` contains `"sales"`  
      - `{{$json.text}}` contains `"jobs"`  
    - *Inputs:* Email data from Gmail Trigger.  
    - *Outputs:*  
      - True branch: email passes filter and proceeds to AI summarization.  
      - False branch: email is ignored.  
    - *Edge cases:*  
      - Case sensitivity might cause missed matches if emails use different casing (e.g., "Sales" vs "sales").  
      - Empty or missing subject/text fields could cause condition failures.  
      - Overlapping keywords or false positives depending on keyword choice.

  - **Ignore Unimportant Email**  
    - *Type:* NoOp (no operation) node  
    - *Role:* Acts as a dead-end for unimportant emails, effectively stopping processing.  
    - *Inputs:* False branch of IF node.  
    - *Outputs:* None.  
    - *Edge cases:* None; this node simply discards data.

#### 2.3 AI Summarization

- **Overview:** Uses an advanced AI agent (GPT-4o-mini) to generate a concise, plain-language summary of the email. The summary mimics Telegram messaging style with emojis and a max length of 300 characters.
- **Nodes Involved:**  
  - Summarize Email with GPT-4o (Langchain Agent node)  
  - OpenAI Chat Model (supporting language model node used by the agent)

- **Node Details:**

  - **Summarize Email with GPT-4o**  
    - *Type:* Langchain Agent (OpenAI wrapper)  
    - *Role:* Sends formatted email content to the AI for summarization with a system prompt defining the task.  
    - *Configuration:*  
      - Input text combines email subject and body:  
        ```
        Email Subject:  {{ $json.subject }}
        Email Body:  {{ $json.text }}
        ```  
      - System prompt instructs the AI to:  
        - Understand various email types (work, personal, financial, etc.)  
        - Generate a short summary (max 300 characters) using plain English and emojis  
        - Highlight key info (payment due, meeting, delivery, alert)  
        - Format output as a clean text summary only, no formatting tags  
      - Prompt includes multiple examples for the AI to model style and tone.  
    - *Inputs:* Email data from IF node’s true branch.  
    - *Outputs:* JSON containing the summary text in `output`.  
    - *Version Requirements:* Uses Langchain Agent node v1.8 with output parser enabled.  
    - *Potential failures:* API rate limits, malformed prompts, network issues, unexpected AI responses, empty email content causing poor summaries.

  - **OpenAI Chat Model**  
    - *Type:* Language model node (gpt-4o-mini)  
    - *Role:* Underlying model accessed by the agent node to generate completions.  
    - *Configuration:* Uses the GPT-4o-mini model variant for cost-effective summarization.  
    - *Credentials:* Requires valid OpenAI API key with access to GPT-4o-mini.  
    - *Inputs:* From agent node as subcomponent.  
    - *Outputs:* Text completions for agent processing.  
    - *Failures:* API key invalidity, model unavailability, network errors.

#### 2.4 Telegram Notification

- **Overview:** Sends the AI-generated email summary as a Telegram message to a configured chat ID.
- **Nodes Involved:**  
  - Send Summary to Telegram (Telegram node)

- **Node Details:**

  - **Send Summary to Telegram**  
    - *Type:* Telegram node (message sender)  
    - *Role:* Posts the generated summary text to a Telegram chat.  
    - *Configuration:*  
      - Text message is set dynamically: `={{ $json.output }}` (the AI summary).  
      - Chat ID specified as `7917193308` (must be user’s or group’s Telegram chat ID).  
      - No additional fields configured.  
    - *Credentials:* Requires Telegram Bot API token with send message rights.  
    - *Inputs:* Summary JSON from AI summarization node.  
    - *Outputs:* None (final node).  
    - *Edge cases:* Invalid chat ID, bot token expired or revoked, Telegram API downtime, message formatting issues.

---

### 3. Summary Table

| Node Name               | Node Type                          | Functional Role                     | Input Node(s)          | Output Node(s)                  | Sticky Note                                                                                                                             |
|-------------------------|----------------------------------|-----------------------------------|-----------------------|--------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------|
| Check for New Emails     | Gmail Trigger                    | Receive new emails from Gmail     | None                  | Important Email Filter          | This automation checks Gmail every minute for new emails (subject + text).                                                              |
| Important Email Filter   | IF                              | Filter emails by keyword importance | Check for New Emails   | Summarize Email with GPT-4o (true branch), Ignore Unimportant Email (false branch) | Checks if subject or body contains keywords like "sales" or "jobs". Modify keywords as needed.                                            |
| Ignore Unimportant Email | NoOp                            | Discard unimportant emails        | Important Email Filter | None                           | Stops processing for unimportant emails.                                                                                                |
| Summarize Email with GPT-4o | Langchain Agent (OpenAI)      | Generate short AI summary         | Important Email Filter | Send Summary to Telegram       | Uses GPT-4o-mini with a system prompt to create a Telegram-style summary of max 300 chars with emojis and plain language.              |
| OpenAI Chat Model        | Language Model (OpenAI GPT-4o-mini) | Underlying AI model for summarization | Summarize Email with GPT-4o (internal) | Summarize Email with GPT-4o (internal) | Model used by agent node to process prompts and generate text.                                                                           |
| Send Summary to Telegram | Telegram                        | Send generated summary to Telegram chat | Summarize Email with GPT-4o | None                           | Sends the AI summary text to Telegram chat ID 7917193308. Ensure bot and chat ID are set correctly.                                       |
| Sticky Note              | Sticky Note                     | Documentation / process explanation | None                  | None                           | Full explanation and node-by-node breakdown with usage notes, sample messages, and modification tips.                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Gmail Trigger node**  
   - Name: `Check for New Emails`  
   - Type: `Gmail Trigger`  
   - Set `Simple` to `false` to get full email content.  
   - Under `Poll Times`, configure to poll every 1 minute (`mode: everyMinute`).  
   - Connect with your Gmail OAuth2 credentials.  
   - No filters configured at this stage.

2. **Add IF node for importance filtering**  
   - Name: `Important Email Filter`  
   - Type: `IF`  
   - Set condition combinator to `OR`.  
   - Add two string contains conditions:  
     - Left value: `={{ $json.subject }}`, contains `"sales"`.  
     - Left value: `={{ $json.text }}`, contains `"jobs"`.  
   - Case sensitivity is enabled; adjust if needed.  
   - Connect input from `Check for New Emails`.

3. **Add NoOp node to discard unimportant emails**  
   - Name: `Ignore Unimportant Email`  
   - Type: `NoOp`  
   - Connect input from the `false` output of `Important Email Filter`.

4. **Add Langchain Agent node for summarization**  
   - Name: `Summarize Email with GPT-4o`  
   - Type: `@n8n/n8n-nodes-langchain.agent`  
   - Set the `promptType` to `define`.  
   - Configure `text` parameter with expression:  
     ```
     Email Subject:  {{ $json.subject }}
     Email Body:  {{ $json.text }}
     ```  
   - Paste the system prompt (copy from the original workflow’s prompt) that instructs the AI to summarize emails in plain language with emojis, max 300 characters, no formatting tags, includes examples.  
   - Enable output parser.  
   - Connect input from the `true` output of `Important Email Filter`.

5. **Add OpenAI Chat Model node for the underlying language model**  
   - Name: `OpenAI Chat Model`  
   - Type: `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
   - Select model `gpt-4o-mini` from the list.  
   - Connect it internally to the Langchain Agent node (usually automatic within agent node setup).  
   - Configure OpenAI API credentials with a valid API key.

6. **Add Telegram node to send the summary**  
   - Name: `Send Summary to Telegram`  
   - Type: `Telegram`  
   - Set `Text` parameter to expression: `={{ $json.output }}` (the summary from the AI).  
   - Set `Chat ID` to your Telegram chat ID (e.g., `7917193308`).  
   - Connect input from the output of `Summarize Email with GPT-4o`.  
   - Configure Telegram API credentials with your bot token that has send message permissions.

7. **Activate workflow**  
   - Save and activate the workflow.  
   - Test by sending emails matching the keywords to your Gmail inbox and verify Telegram messages are received.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                | Context or Link                                                                                                   |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------|
| This workflow is optimized for billing alerts, delivery updates, job or HR emails, and OTP/security messages. Modify keyword filters as needed for your use case.                                                                                                           | Sticky Note content in the workflow.                                                                             |
| Sample Telegram message format: `📦 Your Flipkart order “Bluetooth Speaker” was delivered today. Enjoy!`                                                                                                                                                                    | Example from Sticky Note.                                                                                        |
| Make sure your Gmail OAuth2 and Telegram bot credentials are correctly set and have necessary permissions to avoid authentication errors.                                                                                                                                   | General best practice.                                                                                            |
| The keywords in the filter node are case sensitive; consider adding variations or using regex for more robust matching.                                                                                                                                                     | Potential improvement note.                                                                                       |
| For more advanced email processing, consider expanding the AI prompt or adding additional filtering nodes before summarization.                                                                                                                                             | Workflow extensibility note.                                                                                      |

---

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

---