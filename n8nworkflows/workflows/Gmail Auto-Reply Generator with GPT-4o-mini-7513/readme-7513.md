Gmail Auto-Reply Generator with GPT-4o-mini

https://n8nworkflows.xyz/workflows/gmail-auto-reply-generator-with-gpt-4o-mini-7513


# Gmail Auto-Reply Generator with GPT-4o-mini

---

### 1. Workflow Overview

This workflow automates the generation of polite, context-aware email replies for unread Gmail messages using an AI language model (GPT-4o-mini). It is designed for users who want to handle incoming emails efficiently by automatically drafting replies only when appropriate, such as avoiding spam or irrelevant notifications. The workflow includes the following logical blocks:

- **1.1 Input Reception:** Detects new unread emails in Gmail and retrieves their full content.
- **1.2 Data Preparation:** Extracts essential email fields needed for AI processing.
- **1.3 AI Processing:** Uses a LangChain AI Agent with GPT-4o-mini to decide whether a reply is needed and drafts a response accordingly.
- **1.4 Conditional Action:** Checks the AI's decision and, if a reply is needed, creates a Gmail draft and labels the email thread for follow-up.
- **1.5 Setup & Documentation:** Provides setup instructions and operational notes via sticky notes for user guidance.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:** This block triggers the workflow when a new unread email arrives in the Gmail inbox and fetches the full email details for processing.
- **Nodes Involved:**  
  - Trigger on New Email  
  - Get Unread Email

- **Node Details:**

  - **Trigger on New Email**  
    - Type: Gmail Trigger (polling trigger)  
    - Role: Watches for new unread emails in Gmail, triggering workflow execution every minute.  
    - Configuration: Filters only unread emails; polls every minute.  
    - Inputs: None (trigger node)  
    - Outputs: Email metadata including message ID  
    - Edge Cases: Gmail API quota limits, connectivity errors, or permission issues may cause trigger failures.

  - **Get Unread Email**  
    - Type: Gmail node  
    - Role: Retrieves the full email message by ID passed from the trigger.  
    - Configuration: Uses messageId expression `={{ $json.id }}` from trigger output to fetch email details  
    - Inputs: Output from "Trigger on New Email"  
    - Outputs: Full email JSON including headers, text, subject, threadId, etc.  
    - Edge Cases: API failures, invalid message IDs, or permission issues.

#### 2.2 Data Preparation

- **Overview:** Extracts the minimal required email fields (thread ID, message ID, from address, message text, and subject) for AI processing simplicity and clarity.
- **Nodes Involved:**  
  - Get Required Fields

- **Node Details:**

  - **Get Required Fields**  
    - Type: Set node  
    - Role: Selects and assigns relevant email fields to structured variables for the AI agent.  
    - Configuration: Assigns `threadId`, `id`, `From` (email sender), `message` (email body text), and `subject` from previous node JSON using expressions.  
    - Inputs: Output from "Get Unread Email"  
    - Outputs: Structured JSON with only required fields  
    - Edge Cases: Missing or malformed fields in the email (e.g., empty subject, no text body).

#### 2.3 AI Processing

- **Overview:** Uses an AI language model to analyze the email content, decide if a reply is needed, and compose a short, polite reply if appropriate.
- **Nodes Involved:**  
  - AI Agent (LangChain Agent)  
  - OpenAI (GPT-4o-mini model)  
  - Structured Output (JSON Schema Parser)

- **Node Details:**

  - **AI Agent**  
    - Type: LangChain Agent node  
    - Role: Implements the AI prompt logic for reply decision and drafting.  
    - Configuration:  
      - Prompt instructs the AI to:  
        1. Read the incoming message.  
        2. Determine if a reply is necessary (excluding spam, notifications, irrelevant messages).  
        3. If replying, draft a brief, polite message acknowledging the sender, under 120 words, without false promises.  
      - Output must be valid JSON with `isReply` (boolean) and `response` (string).  
    - Inputs: Output from "Get Required Fields"  
    - Outputs: Raw AI-generated JSON string  
    - Edge Cases: AI misinterpretation, invalid JSON output, model API rate limits, prompt failures.

  - **OpenAI**  
    - Type: LangChain OpenAI Chat node  
    - Role: Provides the GPT-4o-mini language model for AI Agent's prompt completion.  
    - Configuration: Uses GPT-4o-mini model, no additional options.  
    - Inputs: From "AI Agent" as language model provider  
    - Outputs: Language model text responses  
    - Edge Cases: API key invalid/missing, rate limits, network issues.

  - **Structured Output**  
    - Type: Output Parser (LangChain Structured JSON Parser)  
    - Role: Validates and parses AI Agent's JSON output according to the defined schema.  
    - Configuration:  
      - JSON schema requires `isReply` (boolean) and `response` (string).  
    - Inputs: AI Agent raw output  
    - Outputs: Parsed JSON object with validated fields  
    - Edge Cases: Parsing errors if AI returns malformed JSON, schema mismatches.

#### 2.4 Conditional Action

- **Overview:** Based on AI’s decision, this block either creates a Gmail draft with the AI-generated reply or halts if no reply is needed. If a draft is created, the email thread is labeled "Action" for user follow-up.
- **Nodes Involved:**  
  - Can AI Agent Respond? (If node)  
  - Create a draft (Gmail node)  
  - Label Email As "Action" (Gmail node)

- **Node Details:**

  - **Can AI Agent Respond?**  
    - Type: If node  
    - Role: Checks if `isReply` is true in AI Agent output to decide workflow continuation.  
    - Configuration: Condition checks if `{{$json.output.isReply}} == true` (string equality, loose validation)  
    - Inputs: Parsed JSON from "AI Agent" output parser  
    - Outputs:  
      - True branch: Continue to draft creation  
      - False branch: End workflow (no reply)  
    - Edge Cases: Missing or unexpected JSON fields.

  - **Create a draft**  
    - Type: Gmail node  
    - Role: Creates a draft email reply in Gmail with AI-generated text.  
    - Configuration:  
      - Uses message body from AI response: `{{$json.output.response}}`  
      - Uses threadId and subject from "Get Required Fields" for threading and subject preservation  
      - Resource set to "draft" to create draft instead of sending directly  
    - Inputs: True branch from "Can AI Agent Respond?"  
    - Outputs: Draft creation confirmation  
    - Edge Cases: API errors, invalid message formatting, permission issues.

  - **Label Email As "Action"**  
    - Type: Gmail node  
    - Role: Adds a Gmail label ("Action") to the email thread for easy identification.  
    - Configuration:  
      - Adds label with ID `Label_4601850039988562325` to threadId from "Get Required Fields"  
    - Inputs: Output from "Create a draft"  
    - Outputs: Labeling success confirmation  
    - Edge Cases: Invalid label ID, label not existing in Gmail, API errors.

#### 2.5 Setup & Documentation

- **Overview:** Provides user-facing notes including setup instructions and AI reply logic details to assist workflow configuration and customization.
- **Nodes Involved:**  
  - Sticky Note  
  - Sticky Note1

- **Node Details:**

  - **Sticky Note**  
    - Type: Sticky Note  
    - Role: Displays setup instructions  
    - Content:  
      - Connect Gmail and OpenAI credentials before activating the workflow.  
      - Ensure Gmail has a label named `Action`.  
    - Position: Top-left for visibility.

  - **Sticky Note1**  
    - Type: Sticky Note  
    - Role: Explains AI reply logic and customization tips  
    - Content:  
      - AI excludes spam, notifications, irrelevant emails from replies.  
      - Only genuine messages get drafted replies for review.  
      - The AI agent prompt can be customized to reflect brand voice.  
    - Positioned near AI Agent node for contextual reference.

---

### 3. Summary Table

| Node Name                  | Node Type                        | Functional Role                        | Input Node(s)                | Output Node(s)                   | Sticky Note                                                                                           |
|----------------------------|---------------------------------|-------------------------------------|-----------------------------|---------------------------------|-----------------------------------------------------------------------------------------------------|
| Trigger on New Email        | Gmail Trigger                   | Watches for new unread emails        | None                        | Get Unread Email                | 🛠️ Setup Instructions: Connect Gmail and OpenAI credentials before activating. Ensure Gmail label `Action` exists. |
| Get Unread Email            | Gmail                          | Fetches full unread email details    | Trigger on New Email          | Get Required Fields             |                                                                                                     |
| Get Required Fields         | Set                           | Extracts key email fields             | Get Unread Email              | AI Agent                       |                                                                                                     |
| AI Agent                   | LangChain Agent                | Decides if reply needed & drafts it | Get Required Fields           | Can AI Agent Respond?           | ⚠️ Reply Logic: AI excludes spam/notifications. Genuine messages get replies. Customize prompt as needed.  |
| OpenAI                     | LangChain OpenAI Chat          | Provides GPT-4o-mini language model | AI Agent (languageModel input) | AI Agent                      |                                                                                                     |
| Structured Output          | LangChain Output Parser         | Validates AI JSON output              | AI Agent (ai_outputParser)    | AI Agent                       |                                                                                                     |
| Can AI Agent Respond?       | If                            | Checks if AI decided to reply        | AI Agent                     | Create a draft                 |                                                                                                     |
| Create a draft             | Gmail                         | Creates Gmail draft with AI reply    | Can AI Agent Respond? (true) | Label Email As "Action"         |                                                                                                     |
| Label Email As "Action"     | Gmail                         | Adds "Action" label to email thread  | Create a draft                | None                          |                                                                                                     |
| Sticky Note                | Sticky Note                   | Setup instructions display            | None                        | None                          | 🛠️ Setup Instructions: Connect Gmail and OpenAI credentials before activating. Ensure Gmail label `Action` exists. |
| Sticky Note1               | Sticky Note                   | AI reply logic & customization notes | None                        | None                          | ⚠️ Reply Logic: AI excludes spam/notifications. Genuine messages get replies. Customize prompt as needed.  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Gmail Trigger Node**
   - Type: Gmail Trigger  
   - Configure:  
     - Set to trigger on unread emails only (`readStatus`: unread)  
     - Poll interval: every minute  
   - No credentials needed yet; configure Gmail OAuth2 credentials before activation.

2. **Create Gmail Node to Get Email**
   - Type: Gmail  
   - Set operation: `get` message  
   - Message ID: expression `={{ $json.id }}` from trigger node output  
   - Set credentials: Gmail OAuth2

3. **Add Set Node to Extract Required Fields**
   - Type: Set  
   - Add fields:  
     - `threadId` = `={{ $json.threadId }}`  
     - `id` = `={{ $json.id }}`  
     - `From` = `={{ $json.headers.from }}`  
     - `message` = `={{ $json.text }}`  
     - `subject` = `={{ $json.subject }}`

4. **Create LangChain AI Agent Node**
   - Type: LangChain Agent  
   - Prompt:  
     ```
     You are an assistant that helps draft email replies.

     Your task:
     1. Read the email message below:
     "{{ $json.message }}"

     2. Decide if this is an email that requires a reply.  
        - If it's spam, a notification, or irrelevant, set `isReply` to false.  
        - If it's a genuine message that can be replied to, set `isReply` to true.  

     3. If `isReply` is true, draft a short and polite reply that:  
        - Acknowledges the sender.  
        - Is under 120 words.  
        - Does not promise action you cannot take.  

     4. Return the result **only in valid JSON**, no extra text.

     JSON format to return:
     {
       "isReply": true | false,
       "response": "string or empty if isReply is false"
     }
     ```
   - Enable output parsing (expects JSON output).

5. **Create LangChain OpenAI Chat Node**
   - Type: LangChain OpenAI Chat  
   - Model: Select `gpt-4o-mini`  
   - Connect as languageModel input to AI Agent node  
   - Set OpenAI credentials (API key)

6. **Add LangChain Structured Output Parser Node**
   - Type: Output Parser (Structured JSON)  
   - Configure input schema (manual):  
     ```json
     {
       "type": "object",
       "properties": {
         "isReply": {
           "type": "boolean",
           "description": "Whether the email requires a reply"
         },
         "response": {
           "type": "string",
           "description": "The drafted reply text if isReply is true, otherwise empty"
         }
       },
       "required": ["isReply", "response"]
     }
     ```
   - Connect AI Agent’s ai_outputParser output to this node, and then back into AI Agent main output

7. **Add If Node to Check AI Decision**
   - Type: If  
   - Condition: Expression `{{$json.output.isReply}} == true` (string equals, loose validation)  
   - Input from AI Agent output parser

8. **Create Gmail Node to Draft Reply**
   - Type: Gmail  
   - Resource: draft  
   - Parameters:  
     - Message: `{{$json.output.response}}`  
     - Subject: `{{$('Get Required Fields').item.json.subject}}`  
     - Options: use `threadId` from Get Required Fields to keep thread context  
   - Input from If node’s true branch  
   - Set Gmail credentials

9. **Create Gmail Node to Label Email Thread**
   - Type: Gmail  
   - Operation: addLabels  
   - Resource: thread  
   - Parameters:  
     - Label IDs: `Label_4601850039988562325` (ensure label “Action” exists in Gmail)  
     - Thread ID: `{{$('Get Required Fields').item.json.threadId}}`  
   - Input from draft node output  
   - Set Gmail credentials

10. **Add Sticky Notes (Optional but Recommended)**
    - Create one sticky note with setup instructions (connect Gmail and OpenAI, label `Action` must exist)  
    - Create another sticky note near AI Agent explaining the reply logic and how to customize the prompt for brand voice

11. **Connect all nodes in the following order:**

    Trigger on New Email → Get Unread Email → Get Required Fields → AI Agent →  
    → Structured Output Parser → AI Agent output → Can AI Agent Respond? (If) →  
    → True: Create a draft → Label Email As "Action"

---

### 5. General Notes & Resources

| Note Content                                                                                           | Context or Link                                     |
|------------------------------------------------------------------------------------------------------|----------------------------------------------------|
| Connect Gmail and OpenAI credentials before activating the workflow.                                  | Setup Instructions Sticky Note                      |
| Ensure your Gmail account has a label named `Action` to label threads requiring follow-up.           | Setup Instructions Sticky Note                      |
| AI excludes spam, notifications, and irrelevant emails from replies. Only genuine messages get drafted replies. | Reply Logic Sticky Note                             |
| Customize the AI Agent’s prompt to adjust the tone and style of replies to match your brand voice.   | Reply Logic Sticky Note                             |
| The workflow uses GPT-4o-mini model for cost-effective AI assistance while maintaining quality.       | AI Agent Node Configuration                         |

---

Disclaimer: The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.

---