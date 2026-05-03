Automated Email Support with Google Gemini & Human-in-the-Loop Learning

https://n8nworkflows.xyz/workflows/automated-email-support-with-google-gemini---human-in-the-loop-learning-8779


# Automated Email Support with Google Gemini & Human-in-the-Loop Learning

### 1. Workflow Overview

This workflow, titled **Automated Email Support with Google Gemini & Human-in-the-Loop Learning**, automates customer support email handling by combining AI-powered knowledge base querying with human expert intervention to continuously improve the system. It is designed to automatically process incoming support emails, attempt to answer them using an AI-trained knowledge base stored in Google Sheets, and escalate unanswered questions to a human expert. Human replies are then transformed into reusable Q&A pairs and appended to the knowledge base, enabling the AI to learn and improve over time.

The workflow is logically structured into the following blocks:

- **1.1 Input Reception:** Trigger on new incoming emails to the support mailbox.
- **1.2 Support Request Classification:** AI classification of incoming emails to filter support-related requests.
- **1.3 Knowledge Base Preparation:** Retrieval and formatting of existing Q&A data from Google Sheets.
- **1.4 AI Answer Search:** AI attempts to find a relevant answer in the knowledge base and formats a response email.
- **1.5 Conditional Routing:** Branching logic depending on whether an answer was found by the AI.
- **1.6 Automated Reply Sending:** Sending the AI-generated answer directly to the customer.
- **1.7 Human Expert Escalation:** Forwarding the unanswered question to a human expert for response.
- **1.8 Human Answer Processing and Learning:** Transforming the human expert’s reply into a reusable Q&A pair and updating the knowledge base.
- **1.9 No Operation / Workflow Loop:** Ensures workflow continuity and resets.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:** Listens for new incoming emails in the designated Gmail account to start processing.
- **Nodes Involved:**
  - `On New Email Received`
- **Node Details:**
  - **Type:** Gmail Trigger
  - **Role:** Monitors Gmail inbox for new messages every minute.
  - **Configuration:** Polling every minute, no filters applied (captures all incoming emails).
  - **Credentials:** Gmail OAuth2 credential named "Franck".
  - **Input/Output:** No input, outputs email data including text, sender, subject.
  - **Edge Cases:** Gmail API limits or connectivity issues; malformed email content.
  - **Notes:** Starting point of workflow.

#### 1.2 Support Request Classification

- **Overview:** Uses AI to classify incoming emails as support requests or other categories, filtering out irrelevant emails.
- **Nodes Involved:**
  - `Is it a Support Request?`
- **Node Details:**
  - **Type:** Langchain Text Classifier
  - **Role:** Classifies email text into categories: "support" or "other".
  - **Configuration:** Input text from the new email’s body; categories defined explicitly.
  - **Input:** Email text from `On New Email Received`.
  - **Output:** Classification result directing workflow.
  - **Edge Cases:** Misclassification due to ambiguous language; empty or very short emails.
  - **Notes:** Filters workflow to focus only on relevant support emails.

#### 1.3 Knowledge Base Preparation

- **Overview:** Retrieves the existing Q&A knowledge base from Google Sheets and formats it for AI consumption.
- **Nodes Involved:**
  - `Get Knowledge Base`
  - `Format Q&A Pairs for AI`
  - `Combine into Knowledge Context`
- **Node Details:**
  - **Get Knowledge Base:**
    - **Type:** Google Sheets Read
    - **Role:** Reads all rows from sheet “QA Database” containing Q&A pairs.
    - **Configuration:** Document ID and sheet name preset to specific Google Sheet.
    - **Credentials:** Google Sheets OAuth2 credential "Franck".
    - **Edge Cases:** Access denied, empty or malformed sheet.
  - **Format Q&A Pairs for AI:**
    - **Type:** Set Node
    - **Role:** Converts each row into JSON object with "question" and "answer" strings.
    - **Configuration:** Uses JSON string conversion for both fields.
  - **Combine into Knowledge Context:**
    - **Type:** Aggregate Node
    - **Role:** Aggregates all Q&A JSON objects into a single data array for AI input.
    - **Edge Cases:** Empty dataset leads to no knowledge base context.
  
#### 1.4 AI Answer Search

- **Overview:** Core AI logic attempts to find a relevant answer from the knowledge base for the incoming email.
- **Nodes Involved:**
  - `Find Answer with AI`
  - `Structured Output Parser`
- **Node Details:**
  - **Find Answer with AI:**
    - **Type:** Langchain Chain LLM Node
    - **Role:** Takes customer email text and the knowledge base context to generate a JSON output indicating if answer found; either a full HTML email reply or a summarized question for human escalation.
    - **Model:** Connected downstream to Google Gemini 2.5 Pro node (language model).
    - **Prompt:** Detailed instructions to find relevant answers with high confidence or else summarize question.
    - **Input:** Customer email text and knowledge base JSON array.
    - **Output:** JSON object with fields `answerFound`, `responseEmailHtml`, `summarizedQuestion`.
    - **Edge Cases:** AI may produce invalid JSON or low-confidence answers; timeout or API limit errors.
  - **Structured Output Parser:**
    - **Type:** Langchain Output Parser
    - **Role:** Parses AI output JSON strictly to ensure workflow can process results.
    - **Configuration:** JSON schema example provided for validation.
    - **Input:** AI raw output.
    - **Output:** Parsed JSON usable downstream.
    - **Edge Cases:** Parsing errors if AI output malformed.

#### 1.5 Conditional Routing

- **Overview:** Decides the next step based on AI result: send answer or escalate.
- **Nodes Involved:**
  - `Support Answer Found`
- **Node Details:**
  - **Type:** If Node
  - **Role:** Checks if `answerFound` is true.
  - **Input:** Output from `Find Answer with AI`.
  - **Output:** Branch 1 for true (answer found), branch 2 for false (no answer).
  - **Edge Cases:** Missing field or unexpected data type.

#### 1.6 Automated Reply Sending

- **Overview:** Sends the AI-generated response email directly to the customer.
- **Nodes Involved:**
  - `Send AI Answer`
- **Node Details:**
  - **Type:** Gmail Node (Send Email)
  - **Role:** Sends email to original sender with AI answer in HTML format.
  - **Configuration:** Uses sender email from incoming email; subject prepended with “Re:” and “Automated Response”.
  - **Credentials:** Gmail OAuth2 “Franck”.
  - **Input:** AI generated email HTML from previous nodes.
  - **Edge Cases:** Email sending failures, incorrect recipient address.

#### 1.7 Human Expert Escalation

- **Overview:** When AI cannot answer, forwards the summarized question and original email to a designated human expert for manual reply.
- **Nodes Involved:**
  - `Ask Human for Help`
- **Node Details:**
  - **Type:** Gmail Node (Send and Wait for Reply)
  - **Role:** Sends email to expert, waits for reply with expert’s answer.
  - **Configuration:** 
    - Recipient email hardcoded (configurable).
    - Email includes original customer question and details.
    - Response form enabled with instructions.
  - **Credentials:** Gmail OAuth2 “Franck”.
  - **Input:** Summarized question and original email data.
  - **Output:** Expert’s reply text captured for downstream use.
  - **Edge Cases:** Expert delay or no reply; email delivery failure.

#### 1.8 Human Answer Processing and Learning

- **Overview:** Converts human expert reply and original question into a reusable, generic Q&A pair; appends it to the knowledge base.
- **Nodes Involved:**
  - `AI: Create Reusable Q&A`
  - `Structured Output Parser (1)`
  - `Add to Knowledge Base`
- **Node Details:**
  - **AI: Create Reusable Q&A:**
    - **Type:** Langchain Chain LLM Node
    - **Role:** Given original question and expert reply, generates a generalized, polished Q&A pair in JSON.
    - **Model:** Connected to Google Gemini 2.5 Flash.
    - **Prompt:** Extensive instructions to generalize question, clean answer for reuse.
    - **Input:** Original customer question and expert’s reply text.
    - **Output:** JSON with `question` and `answer`.
    - **Edge Cases:** AI generation errors, malformed JSON.
  - **Structured Output Parser (1):**
    - **Type:** Langchain Output Parser
    - **Role:** Validates and parses AI-generated Q&A JSON.
  - **Add to Knowledge Base:**
    - **Type:** Google Sheets Append
    - **Role:** Adds new Q&A pair as a new row to the Google Sheet knowledge base.
    - **Configuration:** Maps “Question” and “Answer” columns.
    - **Credentials:** Google Sheets OAuth2 “Franck”.
    - **Edge Cases:** Sheet access errors, duplicate entries.

#### 1.9 No Operation / Workflow Loop

- **Overview:** Placeholder node to finalize workflow branch without further action.
- **Nodes Involved:**
  - `No Operation, do nothing`
- **Node Details:**
  - **Type:** NoOp Node
  - **Role:** Ends the workflow branch cleanly.
  - **Input:** From `Add to Knowledge Base`.
  - **Output:** Looping back to `Get Knowledge Base` for fresh data.
  - **Edge Cases:** None, safe fallback.

---

### 3. Summary Table

| Node Name               | Node Type                               | Functional Role                                     | Input Node(s)            | Output Node(s)             | Sticky Note                                                                                   |
|-------------------------|---------------------------------------|----------------------------------------------------|--------------------------|----------------------------|-----------------------------------------------------------------------------------------------|
| On New Email Received    | Gmail Trigger                         | Trigger on incoming emails                          | —                        | Is it a Support Request?    | © 2025 Lucas Peyrin; See sticky note (0) with setup instructions and workflow overview        |
| Is it a Support Request? | Langchain Text Classifier             | Classify emails as support or other                 | On New Email Received     | Get Knowledge Base          | See sticky note (9) about classification role and Google Gemini usage                         |
| Get Knowledge Base       | Google Sheets Read                   | Retrieve Q&A knowledge base                          | Is it a Support Request?  | Format Q&A Pairs for AI     | See sticky note (4) about Google Sheets setup                                               |
| Format Q&A Pairs for AI  | Set Node                            | Format sheet rows into JSON Q&A objects             | Get Knowledge Base        | Combine into Knowledge Context | © 2025 Lucas Peyrin                                                                          |
| Combine into Knowledge Context | Aggregate Node                 | Combine all Q&A into data array for AI              | Format Q&A Pairs for AI   | Find Answer with AI         | © 2025 Lucas Peyrin                                                                          |
| Google Gemini 2.5 Pro    | Langchain LLM (Google Gemini)         | AI language model used downstream of input          | —                        | Find Answer with AI (AI Language Model) | See sticky note (3) about Google Gemini API key setup                                        |
| Find Answer with AI      | Langchain Chain LLM                   | Attempts to find answer or summarize question       | Combine into Knowledge Context, Structured Output Parser | Support Answer Found          | Core AI node; See sticky note (6) about prompt customization                                  |
| Structured Output Parser | Langchain Output Parser               | Parse AI JSON output for answer found               | Gemma (AI model output)   | Find Answer with AI         | Ensures strict JSON parsing; see sticky note (8)                                            |
| Support Answer Found     | If Node                             | Branch depending on AI answer found status          | Find Answer with AI       | Send AI Answer / Ask Human for Help | © 2025 Lucas Peyrin                                                                        |
| Send AI Answer           | Gmail Send Node                      | Sends AI answer email to customer                    | Support Answer Found      | —                          | See sticky note (5) about configuring expert email address                                  |
| Ask Human for Help       | Gmail Send & Wait Node               | Sends question to human expert and waits reply      | Support Answer Found      | AI: Create Reusable Q&A     | See sticky note (5) about expert email customization                                        |
| AI: Create Reusable Q&A  | Langchain Chain LLM                  | Creates reusable Q&A pair from expert reply         | Ask Human for Help        | Add to Knowledge Base       | See sticky note (7) on learning component; connected to Google Gemini 2.5 Flash             |
| Structured Output Parser (1) | Langchain Output Parser          | Parses Q&A JSON output                               | AI: Create Reusable Q&A   | Add to Knowledge Base       | See sticky note (8)                                                                          |
| Add to Knowledge Base    | Google Sheets Append                 | Adds new Q&A pair to Google Sheets                   | Structured Output Parser (1) | No Operation, do nothing   | See sticky note (11) about knowledge base updating                                          |
| No Operation, do nothing | No Operation Node                    | Ends workflow branch cleanly                         | Add to Knowledge Base     | Get Knowledge Base          | © 2025 Lucas Peyrin                                                                          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Gmail Trigger Node:**
   - Name: `On New Email Received`
   - Type: Gmail Trigger
   - Configure OAuth2 credentials for Gmail account.
   - Set to poll every minute, no filters.
   
2. **Add Langchain Text Classifier:**
   - Name: `Is it a Support Request?`
   - Input text: `={{ $('On New Email Received').item.json.text }}`
   - Categories: `support`, `other`
   - Connect from `On New Email Received` main output.

3. **Add Google Sheets Read Node:**
   - Name: `Get Knowledge Base`
   - Connect from `Is it a Support Request?` (support branch)
   - Configure Google Sheets OAuth2 credentials.
   - Select your knowledge base Google Sheet.
   - Sheet name: `QA Database`
   - Read all rows (Question and Answer columns).

4. **Add Set Node to Format Q&A:**
   - Name: `Format Q&A Pairs for AI`
   - Use JSON expression:
     ```
     {
       "question": {{ $json.Question.toJsonString() }},
       "answer": {{ $json.Answer.toJsonString() }}
     }
     ```
   - Connect from `Get Knowledge Base`.

5. **Add Aggregate Node:**
   - Name: `Combine into Knowledge Context`
   - Aggregate all formatted Q&A objects into an array.
   - Connect from `Format Q&A Pairs for AI`.

6. **Add Google Gemini 2.5 Pro Node:**
   - Name: `Google Gemini 2.5 Pro`
   - Set model to `models/gemini-2.5-pro`
   - Temperature: 0 (deterministic)
   - Create Google Gemini API credential with your API key.
   - No direct connection to main flow (used as AI language model node).

7. **Add Langchain Chain LLM Node:**
   - Name: `Find Answer with AI`
   - Input: customer email text and knowledge base JSON array from `Combine into Knowledge Context`.
   - Connect AI model input to `Google Gemini 2.5 Pro` node.
   - Use the detailed prompt provided in the original workflow for answering or summarizing.
   - Connect from `Combine into Knowledge Context`.
   - Connect output to parser next.

8. **Add Langchain Output Parser:**
   - Name: `Structured Output Parser`
   - Configure JSON schema to parse AI output with fields:
     - `answerFound` (boolean)
     - `responseEmailHtml` (string or null)
     - `summarizedQuestion` (string or null)
   - Connect from `Find Answer with AI` AI output.

9. **Add If Node:**
   - Name: `Support Answer Found`
   - Condition: `answerFound` is true (`={{ $('Find Answer with AI').last().json.output.answerFound }}`)
   - Connect from `Find Answer with AI` main output.

10. **Add Gmail Send Node (AI Reply):**
    - Name: `Send AI Answer`
    - Send To: `={{ $('On New Email Received').item.json.from.value[0].address }}`
    - Subject: `=Re: {{ $('On New Email Received').item.json.subject }} - Automated Response`
    - Message (HTML): `={{ $json.output.responseEmailHtml }}`
    - Connect from `Support Answer Found` true branch.
    - Use same Gmail OAuth2 credentials.

11. **Add Gmail Send & Wait Node (Human Escalation):**
    - Name: `Ask Human for Help`
    - Send To: expert support email (e.g. `franckramarojohn+expert@gmail.com`)
    - Subject: `[AUTO SUPPORT] New question requiring your expertise`
    - Body: detailed template including original question, customer info, and instructions.
    - Enable response form for free text reply.
    - Connect from `Support Answer Found` false branch.
    - Use same Gmail OAuth2 credentials.

12. **Add Google Gemini 2.5 Flash Node:**
    - Name: `Google Gemini 2.5 Flash`
    - Model: default (or `models/gemini-2.5-flash`)
    - Temperature: default or 0
    - Create and configure API credential.
    - Connect as AI language model for next node.

13. **Add Langchain Chain LLM Node (Q&A Creation):**
    - Name: `AI: Create Reusable Q&A`
    - Input: original question from email and human expert’s reply text.
    - Connect AI input to `Google Gemini 2.5 Flash`.
    - Use detailed prompt for generating reusable Q&A pairs.
    - Connect from `Ask Human for Help` output.

14. **Add Langchain Output Parser:**
    - Name: `Structured Output Parser (1)`
    - Parse AI output with JSON schema expecting `question` and `answer`.
    - Connect from `AI: Create Reusable Q&A`.

15. **Add Google Sheets Append Node:**
    - Name: `Add to Knowledge Base`
    - Append new row with columns:
      - Question: `={{ $json.output.question }}`
      - Answer: `={{ $json.output.answer }}`
    - Connect from `Structured Output Parser (1)`.
    - Use Google Sheets OAuth2 credentials.
    - Target same Google Sheet and sheet name `QA Database`.

16. **Add No Operation Node:**
    - Name: `No Operation, do nothing`
    - Connect from `Add to Knowledge Base`.
    - Connect output back to `Get Knowledge Base` to loop updated data.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                           | Context or Link                                                                                  |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Workflow uses Google Gemini models for AI tasks. To get API keys, go to [Google AI Studio](https://aistudio.google.com/app/apikey), create a new project and API key, then configure credentials in n8n nodes.                           | See Sticky Note (3) in workflow                                                                |
| Google Sheet knowledge base must have a sheet named `QA Database` with two columns: `Question` and `Answer`. This sheet is used to store and retrieve Q&A pairs for AI reference and learning.                                         | See Sticky Note (4) in workflow                                                                |
| The human expert email address is configurable in the `Ask Human for Help` node. Update this to your actual support expert’s email to forward escalated questions.                                                                     | See Sticky Note (5) in workflow                                                                |
| AI prompts are carefully designed to generalize customer questions and produce polished, re-usable answers suitable for direct customer communication. These are central to the workflow's success and may need tailoring to your domain. | See Sticky Notes (6) and (7) for prompt explanations and customization guidance.                |
| The workflow is designed for continuous operation and learning. Activating the workflow will start automatic processing of incoming emails, AI answering, and human-in-the-loop knowledge base updates.                              | See Sticky Note (10) about activation                                                          |
| For support, coaching, or custom workflow building, contact Lucas Peyrin via the unified AI-powered contact form linked in the workflow notes.                                                                                      | [Get in Touch Here](https://template.workflows.ac?source=AI%20Email%20Support&ref=Franck)      |

---

**Disclaimer:** The text and data described originate exclusively from an automated workflow created with n8n, respecting all content policies. No illegal or protected data is included. All handled data is legal and public.