Generate Context-Aware Gmail Replies with Anthropic Sonnet 4.5 and Email Classification

https://n8nworkflows.xyz/workflows/generate-context-aware-gmail-replies-with-anthropic-sonnet-4-5-and-email-classification-9193


# Generate Context-Aware Gmail Replies with Anthropic Sonnet 4.5 and Email Classification

### 1. Workflow Overview

This workflow automates the generation of context-aware Gmail replies by analyzing entire email threads using AI. It targets users who want to streamline their email responses by leveraging AI models to classify incoming emails and draft replies that consider the full conversation context.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Detects new incoming emails via a Gmail trigger.
- **1.2 Email Classification:** Classifies incoming emails to decide if they warrant a reply.
- **1.3 Thread Retrieval and Processing:** Retrieves the full Gmail thread for contextual understanding and normalizes email data.
- **1.4 AI Reply Generation:** Uses AI language models (OpenAI and Anthropic Sonnet 4.5) to analyze the email thread and draft a reply.
- **1.5 Draft Creation in Gmail:** Creates a draft reply in Gmail with the AI-generated content.
- **1.6 Workflow Management and Merging:** Merges multiple data streams and controls flow between nodes.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
This block listens for new incoming Gmail messages using polling and triggers the workflow upon detection.

**Nodes Involved:**  
- Gmail Trigger

**Node Details:**

- **Gmail Trigger**  
  - Type: Trigger node that polls Gmail API for new emails every minute.  
  - Configuration: Polls Gmail every minute; no custom filters set.  
  - Credentials: Uses OAuth2 credentials for a Gmail account named "n3w.it".  
  - Input/Output: No input; outputs new email metadata including threadId, From, Cc, and Subject.  
  - Edge cases: Possible OAuth token expiration or API rate limits; network issues causing missed triggers.  
  - Version: 1.3

---

#### 1.2 Email Classification

**Overview:**  
Classifies the incoming email snippet to determine if the message should be responded to by the AI assistant.

**Nodes Involved:**  
- Email Classifier  
- OpenAI Chat Model1 (used internally by Email Classifier)

**Node Details:**

- **Email Classifier**  
  - Type: AI text classification node using Langchain interface with OpenAI.  
  - Configuration: Classifies email snippets into categories, with a fallback to "discard" if uncertain. The category of interest is "ok" indicating emails deserving a response. Auto-fixing of output enabled to ensure valid JSON.  
  - Input: Email snippet from trigger node JSON.  
  - Output: JSON classification result indicating whether to proceed.  
  - Edge cases: Classification errors if snippet is missing or malformed; API limits or errors from OpenAI.  
  - Version: 1.1  
  - Sub-workflow: Utilizes OpenAI Chat Model1 node internally.

- **OpenAI Chat Model1**  
  - Type: Language model node using OpenAI GPT-5 Nano.  
  - Configuration: Uses GPT-5 Nano model for classification.  
  - Credentials: OpenAI API key ("Eure" account).  
  - Input: Text to classify passed from Email Classifier.  
  - Output: Classification results back to Email Classifier.  
  - Edge cases: OpenAI API rate limiting, authentication errors, or transient failures.  
  - Version: 1.2

---

#### 1.3 Thread Retrieval and Processing

**Overview:**  
Fetches the entire Gmail thread corresponding to the incoming email and normalizes the email messages for AI processing.

**Nodes Involved:**  
- Get a thread  
- Code  
- Merge

**Node Details:**

- **Get a thread**  
  - Type: Gmail resource node to retrieve full email thread details.  
  - Configuration: Retrieves messages for the threadId received from the trigger. Returns only messages (not metadata).  
  - Credentials: Same Gmail OAuth2 credentials as trigger.  
  - Input: threadId from Gmail Trigger node.  
  - Output: Array of messages in the thread.  
  - Edge cases: Thread ID invalid or deleted; API limits; network failures.  
  - Version: 2.1

- **Code**  
  - Type: JavaScript code node.  
  - Configuration: Normalizes the retrieved thread messages into a structured array containing snippet, from, and to fields for each email. Handles cases when input is single object, array, or empty. Outputs a single object with property `data` containing normalized emails.  
  - Inputs: Raw thread messages from Get a thread node.  
  - Outputs: JSON with property `data` holding normalized emails array.  
  - Edge cases: Unexpected data structure; empty thread; missing fields.  
  - Version: 2

- **Merge**  
  - Type: Merge node combining inputs.  
  - Configuration: Combines all inputs into one single output, combining the trigger item and AI response.  
  - Inputs: From Gmail Trigger and Replying email Agent outputs.  
  - Outputs: Combined data passed to Create a draft node.  
  - Edge cases: Mismatch in item counts, timing issues.  
  - Version: 3.2

---

#### 1.4 AI Reply Generation

**Overview:**  
Uses AI models to analyze the normalized conversation history and draft an HTML email reply.

**Nodes Involved:**  
- Replying email Agent  
- Anthropic Sonnet 4.5

**Node Details:**

- **Replying email Agent**  
  - Type: Langchain agent node specialized for email reply drafting.  
  - Configuration: Receives JSON stringified conversation data. Uses a system message instructing it to craft expert replies strictly based on conversation history. Specifies output must be valid HTML email content only, no explanations or notes, placeholders for missing data.  
  - Input: Normalized emails from Code node (via Merge).  
  - Output: Draft email content as HTML string.  
  - Edge cases: AI hallucinations, incomplete data, malformed JSON input; API errors.  
  - Version: 2.2  
  - Sub-workflow: Uses Anthropic Sonnet 4.5 as underlying AI model.

- **Anthropic Sonnet 4.5**  
  - Type: AI chat language model node using Anthropic's Claude Sonnet 4.5.  
  - Configuration: Selected model "claude-sonnet-4-5-20250929."  
  - Credentials: Anthropic API key.  
  - Input: Passed from Replying email Agent node.  
  - Output: AI-generated email reply content.  
  - Edge cases: API limits, auth failures, model errors.  
  - Version: 1.3

---

#### 1.5 Draft Creation in Gmail

**Overview:**  
Creates a draft email in Gmail with the AI-generated reply, in the context of the original email thread.

**Nodes Involved:**  
- Create a draft

**Node Details:**

- **Create a draft**  
  - Type: Gmail node for creating draft emails.  
  - Configuration:  
    - Sends to original sender (from Gmail Trigger).  
    - CC list passed if present.  
    - Subject copied from original email.  
    - Uses threadId to associate draft with existing conversation.  
    - Email content set to AI-generated HTML output.  
  - Credentials: Same Gmail OAuth2 credentials.  
  - Input: AI reply from Merge node; original email metadata from Gmail Trigger.  
  - Output: Draft creation confirmation.  
  - Edge cases: Gmail API quota limits; invalid email addresses; draft creation failures.  
  - Version: 2.1

---

### 3. Summary Table

| Node Name           | Node Type                                | Functional Role                          | Input Node(s)          | Output Node(s)     | Sticky Note                                                                                                                       |
|---------------------|----------------------------------------|----------------------------------------|------------------------|--------------------|----------------------------------------------------------------------------------------------------------------------------------|
| Gmail Trigger       | n8n-nodes-base.gmailTrigger             | Detects incoming emails                 | —                      | Merge, Email Classifier | ## STEP 1<br>Gmail OAuth2 Credentials required for this and subsequent Gmail nodes; OpenAI and Anthropic API credentials needed. |
| Email Classifier    | @n8n/n8n-nodes-langchain.textClassifier| Classifies emails to decide reply      | Gmail Trigger           | Get a thread       |                                                                                                                                |
| OpenAI Chat Model1  | @n8n/n8n-nodes-langchain.lmChatOpenAi  | Provides classification AI model       | Email Classifier (ai_languageModel) | Email Classifier |                                                                                                                                |
| Get a thread       | n8n-nodes-base.gmail                    | Retrieves full Gmail thread             | Email Classifier        | Code               |                                                                                                                                |
| Code                | n8n-nodes-base.code                     | Normalizes thread messages for AI input| Get a thread            | Replying email Agent|                                                                                                                                |
| Replying email Agent| @n8n/n8n-nodes-langchain.agent          | Drafts AI-generated HTML email reply   | Code                   | Merge              |                                                                                                                                |
| Anthropic Sonnet 4.5| @n8n/n8n-nodes-langchain.lmChatAnthropic| Provides main AI language model        | Replying email Agent (ai_languageModel) | Replying email Agent |                                                                                                                                |
| Merge               | n8n-nodes-base.merge                    | Combines trigger and AI reply data     | Gmail Trigger, Replying email Agent | Create a draft   |                                                                                                                                |
| Create a draft      | n8n-nodes-base.gmail                    | Creates draft reply in Gmail            | Merge                   | —                  |                                                                                                                                |
| Sticky Note         | n8n-nodes-base.stickyNote               | Informational notes                     | —                      | —                  | ## AI-Powered Gmail Assistant: send replies by analyzing Thread ID with Sonnet 4.5<br>This workflow automates analysis and drafting of replies. |
| Sticky Note1        | n8n-nodes-base.stickyNote               | Informational notes                     | —                      | —                  | ## STEP 1<br>Credentials setup instructions for Gmail, OpenAI, and Anthropic APIs.                                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Gmail Trigger node**  
   - Type: Gmail Trigger  
   - Poll interval: every minute  
   - Credentials: Set up Gmail OAuth2 credentials for your Gmail account (e.g., "n3w.it").  
   - No filters configured to listen to all incoming emails.

2. **Create Email Classifier node**  
   - Type: Langchain Text Classifier  
   - Input Text: `={{ $json.snippet }}` from Gmail Trigger node  
   - Categories: Set one category named "ok" meaning emails deserving response  
   - System prompt: Instruct classifier to output JSON only with classification, no explanation  
   - Enable fallback: discard  
   - Enable auto fixing to ensure JSON validity  
   - Credentials: OpenAI API key configured (e.g., GPT-5 Nano model)  

3. **Create OpenAI Chat Model node**  
   - Type: Langchain OpenAI Chat Model  
   - Model: GPT-5 Nano  
   - Credentials: Use OpenAI API key  
   - This node is connected as AI model for Email Classifier node.

4. **Create Get a thread node**  
   - Type: Gmail resource node  
   - Resource: Thread  
   - Operation: Get  
   - Thread ID: `={{ $json.threadId }}` passed from Email Classifier node output  
   - Return only messages: Enabled  
   - Credentials: Same Gmail OAuth2 as trigger

5. **Create Code node**  
   - Type: Code (JavaScript) node  
   - Purpose: Normalize thread messages into an array of objects with snippet, from, and to fields  
   - Use provided JavaScript code to handle various input structures and output a JSON property "data" containing the array

6. **Create Anthropic Sonnet 4.5 node**  
   - Type: Langchain Anthropic Chat Model  
   - Model: Claude Sonnet 4.5 (select version "claude-sonnet-4-5-20250929")  
   - Credentials: Anthropic API key

7. **Create Replying email Agent node**  
   - Type: Langchain Agent  
   - Input text: `={{ JSON.stringify($json.data) }}` from Code node  
   - System message prompt:  
     ```
     You are an expert in replying to emails that arrive in my inbox.  
     You will be provided with the conversation history.  
     Analyze it carefully and draft a reply.  

     - Do not invent information.  
     - If some variables are missing or unknown, insert clear placeholders like {{variable}}.  
     - Your output must be ONLY the email reply in valid HTML format, without any preamble, notes, or explanations.
     ```  
   - Prompt type: Define  
   - Connect Anthropic Sonnet 4.5 node as AI language model for this agent

8. **Create Merge node**  
   - Type: Merge node  
   - Mode: Combine all inputs  
   - Combine by: Combine all  
   - Inputs:  
     - From Gmail Trigger node (main output)  
     - From Replying email Agent node (main output)  

9. **Create Create a draft node**  
   - Type: Gmail node  
   - Resource: Draft  
   - Operation: Create  
   - Subject: `={{ $('Gmail Trigger').item.json.Subject }}`  
   - Message: `={{ $json.output }}` from Merge node output (AI reply)  
   - Send To: `={{ $('Gmail Trigger').item.json.From }}`  
   - CC List: `={{ $ifEmpty($('Gmail Trigger').item.json.Cc, '') }}`  
   - Thread ID: `={{ $('Gmail Trigger').item.json.threadId }}`  
   - Email Type: HTML  
   - Credentials: Same Gmail OAuth2 as others

10. **Connect Nodes**  
    - Gmail Trigger → Merge (input 1)  
    - Gmail Trigger → Email Classifier  
    - Email Classifier → Get a thread  
    - Get a thread → Code  
    - Code → Replying email Agent  
    - Replying email Agent → Merge (input 2)  
    - Merge → Create a draft  

11. **Add Sticky Notes for Documentation** (optional)  
    - Add notes describing workflow purpose and credential requirements for clarity.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
|--------------|-----------------|
| This workflow automates analyzing Gmail threads and drafting AI-powered replies using Anthropic Sonnet 4.5. | See first sticky note for overview. |
| Gmail OAuth2 credentials must be configured consistently for trigger, thread retrieval, and draft creation nodes. | Sticky Note1 details credential setup. |
| OpenAI API credentials are required for the email classification step using GPT-5 Nano. | Sticky Note1. |
| Anthropic API credentials are required for generating draft replies with Claude Sonnet 4.5. | Sticky Note1. |

---

**Disclaimer:**  
The provided text is exclusively derived from an automated workflow created with n8n, a tool for integration and automation. This processing strictly complies with applicable content policies and contains no illegal, offensive, or protected material. All data handled is legal and public.