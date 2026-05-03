Generate GEO-optimized support replies from Gmail to Gmail and Slack

https://n8nworkflows.xyz/workflows/generate-geo-optimized-support-replies-from-gmail-to-gmail-and-slack-11077


# Generate GEO-optimized support replies from Gmail to Gmail and Slack

---

### 1. Workflow Overview

This workflow automates the generation of geographically optimized customer support email replies using Gmail, OpenAI AI models, and Slack notifications. It targets customer service teams that want to automatically classify incoming emails, detect the customer's geographic region (GEO), and produce region-specific, tone-appropriate responses. It simultaneously routes non-support queries for manual review via Slack.

The workflow logic is divided into the following blocks:

- **1.1 Gmail Input Reception:** Listens for unread incoming emails in Gmail inbox.
- **1.2 Email Data Extraction:** Extracts key email fields (sender, subject, message snippet) for AI processing.
- **1.3 Email Classification:** Uses AI to determine if the email is a customer support query or not.
- **1.4 Conditional Routing:** Routes the email based on classification result (support query or manual review).
- **1.5 GEO Identification:** If support query, AI infers customer GEO region from email content.
- **1.6 GEO-Optimized Reply Generation:** AI produces a ready-to-send support reply with tone and content tailored to the GEO.
- **1.7 Formatting and Sending:** Formats AI output, sends the reply via Gmail, and posts a notification to Slack.
- **1.8 Manual Review Notification:** Sends Slack alerts for emails requiring manual human review.
- **1.9 Error Handling:** Captures workflow errors and sends alerts to Slack for debugging.

---

### 2. Block-by-Block Analysis

#### 1.1 Gmail Input Reception

- **Overview:**  
  Polls unread emails from Gmail inbox every minute and triggers the workflow with raw email data.

- **Nodes Involved:**  
  - Gmail Trigger

- **Node Details:**  
  - **Gmail Trigger**  
    - Type: Trigger node for Gmail  
    - Configuration: Filters set to label "INBOX" and unread status; polling every minute  
    - Credentials: OAuth2 Gmail credentials required  
    - Inputs: None (trigger)  
    - Outputs: Raw Gmail message JSON  
    - Failures: Possible auth expiration, API quota limits, network timeouts  
    - Version: 1.3

#### 1.2 Email Data Extraction

- **Overview:**  
  Extracts sender email, subject, and message snippet into simplified fields for downstream AI nodes.

- **Nodes Involved:**  
  - Extract Email Data

- **Node Details:**  
  - **Extract Email Data**  
    - Type: Set node  
    - Configuration: Assigns three string fields from incoming JSON:  
      - `customer_email` = From  
      - `customer_subject` = Subject  
      - `customer_message` = snippet  
    - Inputs: Gmail Trigger raw output  
    - Outputs: Simplified JSON with extracted fields  
    - Version: 3.3  
    - Edge Cases: Missing fields in email, malformed JSON  

#### 1.3 Email Classification

- **Overview:**  
  Classifies incoming email using AI to detect if it is a customer support query and assigns a category with reason.

- **Nodes Involved:**  
  - OpenAI Chat Model - GPT-4o-mini2  
  - Memory - Conversation Buffer2  
  - Output Parser - Structured JSON2  
  - AI Agent - Email Classification

- **Node Details:**  
  - **OpenAI Chat Model - GPT-4o-mini2**  
    - Type: OpenAI chat model (GPT-4o-mini)  
    - Configuration: Standard chat model call without special options  
    - Credentials: OpenAI API credentials  
    - Inputs: Email data from Extract Email Data  
    - Outputs: AI raw chat response  
    - Version: 1.2  
    - Failures: API rate limits, invalid prompt, network errors  

  - **Memory - Conversation Buffer2**  
    - Type: LangChain memory buffer to maintain session context using sessionKey "GEO Defined"  
    - Inputs: Chat model output (ai_languageModel)  
    - Outputs: Passes memory forward  
    - Version: 1.3  

  - **Output Parser - Structured JSON2**  
    - Type: LangChain output parser enforcing strict JSON schema:  
      ```json
      {
        "is_query": true,
        "category": "support",
        "reason": "The email asks about shipping availability"
      }
      ```  
    - Inputs: AI chat output  
    - Outputs: Parsed JSON for conditional logic  
    - Version: 1.3  
    - Edge Cases: Invalid or malformed AI response breaking parser  

  - **AI Agent - Email Classification**  
    - Type: LangChain AI agent node that defines classification prompt and system instructions  
    - Configuration: Prompt instructs to classify email and return JSON only  
    - Inputs: Extracted email fields  
    - Outputs: Parsed JSON classification result  
    - Version: 2.1  
    - Failures: Incorrect prompt formatting, AI hallucination  

#### 1.4 Conditional Routing

- **Overview:**  
  Routes workflow execution based on classification result; processes support queries or sends non-queries for manual review.

- **Nodes Involved:**  
  - If

- **Node Details:**  
  - **If**  
    - Type: Conditional node  
    - Configuration: Checks if `is_query` equals true (boolean strict equality)  
    - Inputs: Output of AI Agent - Email Classification  
    - Outputs:  
      - True branch → AI Agent - GEO Identifier  
      - False branch → Manual Review Slack notification  
    - Version: 2.2  
    - Edge Cases: Missing `is_query` field, parsing errors causing undefined behavior  

#### 1.5 GEO Identification

- **Overview:**  
  AI agent analyzes email content and sender info to infer customer's geographic region (US, UK, India, Canada, EU, Australia, UAE, Unknown).

- **Nodes Involved:**  
  - OpenAI Chat Model - GPT-4o-mini  
  - Memory - Conversation Buffer  
  - Output Parser - Structured JSON  
  - AI Agent - GEO Identifier

- **Node Details:**  
  - **OpenAI Chat Model - GPT-4o-mini**  
    - Type: OpenAI chat model GPT-4o-mini  
    - Configuration: Standard chat call  
    - Credentials: OpenAI API  
    - Inputs: Email data from Extract Email Data  
    - Outputs: Raw AI response  
    - Version: 1.2  

  - **Memory - Conversation Buffer**  
    - Type: Memory buffer session keyed "GEO Defined"  
    - Inputs: AI model output  
    - Outputs: Passes memory forward  
    - Version: 1.3  

  - **Output Parser - Structured JSON**  
    - Type: Enforces JSON response with schema:  
      ```json
      {
        "geo": "Canada"
      }
      ```  
    - Inputs: AI raw output  
    - Outputs: Parsed GEO JSON  
    - Version: 1.3  

  - **AI Agent - GEO Identifier**  
    - Type: AI agent with system instructions to respond only in valid JSON with one GEO label  
    - Inputs: Extracted email data  
    - Outputs: Parsed GEO JSON  
    - Version: 2.1  
    - Edge Cases: Ambiguous GEO hints, AI hallucinations, invalid JSON  

#### 1.6 GEO-Optimized Reply Generation

- **Overview:**  
  Generates a ready-to-send customer support reply email customized with regional tone and rules, returning structured JSON with recipient, subject, reply body, GEO used, and summary.

- **Nodes Involved:**  
  - OpenAI Chat Model - GPT-4o-mini1  
  - Memory - Conversation Buffer1  
  - Output Parser - Structured JSON1  
  - AI Agent - GEO-Optimized Support

- **Node Details:**  
  - **OpenAI Chat Model - GPT-4o-mini1**  
    - Type: OpenAI GPT-4o-mini chat model  
    - Credentials: OpenAI API  
    - Inputs: GEO info and extracted email data  
    - Outputs: Raw AI reply text  
    - Version: 1.2  

  - **Memory - Conversation Buffer1**  
    - Type: Memory buffer session keyed "GEO Defined"  
    - Inputs: AI model output  
    - Outputs: Passes memory forward  
    - Version: 1.3  

  - **Output Parser - Structured JSON1**  
    - Type: Enforces strict JSON schema for reply:  
      ```json
      {
        "to": "laura@example.com",
        "subject": "Re: Shipping to Canada?",
        "reply": "Hello Laura, ...",
        "geo_used": "Canada",
        "summary": "Provided shipping info and fees for a customer in Canada."
      }
      ```  
    - Inputs: Raw AI output  
    - Outputs: Parsed reply JSON  
    - Version: 1.3  
    - Edge Cases: AI output invalid JSON, missing fields  

  - **AI Agent - GEO-Optimized Support**  
    - Type: AI agent node with system message defining tone rules and output format  
    - Inputs: GEO JSON, extracted email data  
    - Outputs: Parsed structured JSON reply  
    - Version: 2.1  
    - Failures: AI hallucination, invalid JSON output  
    - Important: Strict JSON output enforced to avoid downstream failures  

#### 1.7 Formatting and Sending

- **Overview:**  
  Formats AI-generated reply fields into simple keys, sends the reply email via Gmail, and posts a notification message to Slack summarizing the interaction.

- **Nodes Involved:**  
  - Format Data For Email  
  - Send a message (Gmail)  
  - Notify Slack

- **Node Details:**  
  - **Format Data For Email**  
    - Type: Set node  
    - Configuration: Maps AI agent output fields to simple keys: To, Reply, Geo Used, Summary, Subject  
    - Inputs: AI Agent - GEO-Optimized Support output  
    - Outputs: Formatted JSON for sending and notification  
    - Version: 3.4  

  - **Send a message**  
    - Type: Gmail node (send email)  
    - Configuration: Sends email using To, Subject, and Message from formatted data; plain text email  
    - Credentials: Gmail OAuth2  
    - Inputs: Formatted email data  
    - Outputs: None (terminal)  
    - Version: 2.1  
    - Failures: Sending permissions, quota limits, invalid email address  

  - **Notify Slack**  
    - Type: Slack node (send message)  
    - Configuration: Sends notification to specified Slack channel with templated text including recipient, GEO used, and summary  
    - Credentials: Slack API OAuth2  
    - Inputs: Formatted email data  
    - Outputs: None (terminal)  
    - Version: 2.1  

#### 1.8 Manual Review Notification

- **Overview:**  
  For emails classified as non-support queries, sends a Slack notification alerting human agents for manual review and triage.

- **Nodes Involved:**  
  - Manual Review (Slack)

- **Node Details:**  
  - **Manual Review**  
    - Type: Slack node  
    - Configuration: Sends message to Slack channel with classification category and reason, requesting manual check  
    - Credentials: Slack API OAuth2  
    - Inputs: Classification JSON from AI Agent - Email Classification (false branch of If node)  
    - Outputs: None (terminal)  
    - Version: 2.1  

#### 1.9 Error Handling

- **Overview:**  
  Captures any workflow execution errors and sends an alert to Slack with node name, error message, and timestamp for rapid debugging.

- **Nodes Involved:**  
  - Error Handler Trigger  
  - Slack: Send Error Alert

- **Node Details:**  
  - **Error Handler Trigger**  
    - Type: Special trigger node activated on workflow errors  
    - Inputs: Workflow error events  
    - Outputs: Error JSON  

  - **Slack: Send Error Alert**  
    - Type: Slack node  
    - Configuration: Sends error details (node name, message, timestamp) to Slack channel  
    - Credentials: Slack OAuth2 (different credential than main Slack)  
    - Inputs: Error Handler Trigger output  
    - Outputs: None  
    - Version: 2.3  

---

### 3. Summary Table

| Node Name                     | Node Type                                  | Functional Role                         | Input Node(s)                | Output Node(s)                | Sticky Note                                                                                      |
|-------------------------------|--------------------------------------------|---------------------------------------|-----------------------------|------------------------------|------------------------------------------------------------------------------------------------|
| Gmail Trigger                 | Gmail Trigger                             | Listen for unread Gmail inbox emails  | None                        | Extract Email Data            | Polls unread INBOX emails and passes raw message data into the workflow.                        |
| Extract Email Data            | Set Node                                  | Extract key email fields               | Gmail Trigger               | AI Agent - Email Classification | Extracts sender, subject, and snippet to prepare inputs for classification.                     |
| OpenAI Chat Model - GPT-4o-mini2 | OpenAI Chat Model (GPT-4o-mini)            | AI classification language model       | AI Agent - Email Classification (ai_languageModel) | Memory - Conversation Buffer2    |                                                                                                |
| Memory - Conversation Buffer2 | LangChain Memory Buffer                   | Maintain session context for classification | OpenAI Chat Model - GPT-4o-mini2 | AI Agent - Email Classification (ai_memory) |                                                                                                |
| Output Parser - Structured JSON2 | LangChain Output Parser                   | Enforce JSON schema for classification | Memory - Conversation Buffer2 | AI Agent - Email Classification (ai_outputParser) |                                                                                                |
| AI Agent - Email Classification | LangChain AI Agent                        | Classify email as support query or not | Extract Email Data          | If                           | Determines if the message is a customer query and assigns a category and reason.               |
| If                           | If Node                                   | Route based on classification result  | AI Agent - Email Classification | AI Agent - GEO Identifier (true), Manual Review (false) |                                                                                                |
| OpenAI Chat Model - GPT-4o-mini | OpenAI Chat Model (GPT-4o-mini)            | AI language model for GEO identification | AI Agent - GEO Identifier (ai_languageModel) | Memory - Conversation Buffer      |                                                                                                |
| Memory - Conversation Buffer  | LangChain Memory Buffer                   | Maintain session context for GEO ID   | OpenAI Chat Model - GPT-4o-mini | AI Agent - GEO Identifier (ai_memory) |                                                                                                |
| Output Parser - Structured JSON | LangChain Output Parser                   | Enforce JSON schema for GEO response  | Memory - Conversation Buffer | AI Agent - GEO Identifier (ai_outputParser) |                                                                                                |
| AI Agent - GEO Identifier     | LangChain AI Agent                        | Identify customer GEO from email data | Extract Email Data          | AI Agent - GEO-Optimized Support | AI infers customer GEO from email content, address hints, and language cues.                   |
| OpenAI Chat Model - GPT-4o-mini1 | OpenAI Chat Model (GPT-4o-mini)            | AI language model for GEO-optimized reply | AI Agent - GEO-Optimized Support (ai_languageModel) | Memory - Conversation Buffer1     |                                                                                                |
| Memory - Conversation Buffer1 | LangChain Memory Buffer                   | Maintain session context for reply generation | OpenAI Chat Model - GPT-4o-mini1 | AI Agent - GEO-Optimized Support (ai_memory) |                                                                                                |
| Output Parser - Structured JSON1 | LangChain Output Parser                   | Enforce JSON schema for reply output  | Memory - Conversation Buffer1 | AI Agent - GEO-Optimized Support (ai_outputParser) |                                                                                                |
| AI Agent - GEO-Optimized Support | LangChain AI Agent                        | Generate GEO-optimized support reply  | AI Agent - GEO Identifier   | Format Data For Email         | Generates a ready-to-send reply with tone and content tailored to the detected GEO.            |
| Format Data For Email         | Set Node                                  | Format AI reply fields for sending    | AI Agent - GEO-Optimized Support | Send a message, Notify Slack  | Formats AI output for Gmail and sends the reply; also stores summary and GEO used.            |
| Send a message                | Gmail Send Node                           | Send reply email via Gmail             | Format Data For Email       | None                         |                                                                                                |
| Notify Slack                 | Slack Node                                | Send notification of reply to Slack   | Format Data For Email       | None                         | Sends success confirmations and manual-review alerts to the selected Slack channel.           |
| Manual Review                | Slack Node                                | Notify Slack channel for manual review | If (false branch)           | None                         | Routes non-query or uncertain items to Slack for human review and triage.                      |
| Error Handler Trigger        | Error Trigger                             | Trigger on workflow errors             | None                       | Slack: Send Error Alert       | Captures workflow failures and sends details to Slack for quick debugging.                    |
| Slack: Send Error Alert      | Slack Node                                | Send error alerts to Slack             | Error Handler Trigger       | None                         |                                                                                                |
| Sticky Note                  | Sticky Note                               | Workflow overview and instructions     | None                       | None                         | Real-Time GEO-Optimized Answer Engine – Overview; see detailed content in workflow notes.     |
| Sticky Note1                 | Sticky Note                               | Gmail trigger explanation               | None                       | None                         | Gmail Trigger polls unread INBOX emails and passes raw message data into the workflow.        |
| Sticky Note2                 | Sticky Note                               | GEO-Optimized support reply explanation | None                       | None                         | GEO-Optimized Support Reply generates ready-to-send reply tailored by GEO.                    |
| Sticky Note3                 | Sticky Note                               | Slack notifications explanation         | None                       | None                         | Slack sends success confirmations and manual-review alerts to the selected channel.           |
| Sticky Note4                 | Sticky Note                               | Format & send email explanation         | None                       | None                         | Formats AI output for Gmail and sends reply; stores summary and GEO used.                     |
| Sticky Note5                 | Sticky Note                               | Strict JSON requirement warning         | None                       | None                         | All AI agents MUST return valid JSON exactly matching the parser schema.                      |
| Sticky Note6                 | Sticky Note                               | GEO Identifier explanation               | None                       | None                         | AI infers customer GEO from email content, address hints, and language cues.                  |
| Sticky Note7                 | Sticky Note                               | Manual review explanation                | None                       | None                         | Routes non-query or uncertain items to Slack for human review and triage.                     |
| Sticky Note8                 | Sticky Note                               | Email classification explanation        | None                       | None                         | Determines if the message is a customer query and assigns category and reason.                |
| Sticky Note9                 | Sticky Note                               | Email data extraction explanation       | None                       | None                         | Extracts sender, subject, snippet to prepare inputs for classification.                       |
| Error Handling              | Sticky Note                               | Error handling instructions              | None                       | None                         | Captures workflow failures and sends details to Slack for quick debugging.                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Gmail Trigger node:**  
   - Type: Gmail Trigger  
   - Configure to poll unread emails in the "INBOX" label every minute  
   - Set Gmail OAuth2 credentials  
   - Position at start of workflow

2. **Create Set node "Extract Email Data":**  
   - Input: Gmail Trigger output  
   - Assign fields:  
     - `customer_email` = `{{$json.From}}`  
     - `customer_subject` = `{{$json.Subject}}`  
     - `customer_message` = `{{$json.snippet}}`  
   - Version: 3.3

3. **Create AI Agent - Email Classification block:**  
   a. Add OpenAI Chat Model node (GPT-4o-mini)  
      - Model: gpt-4o-mini  
      - Credentials: OpenAI API  
   b. Add Memory Buffer node (Conversation Buffer2)  
      - SessionKey: `"GEO Defined"`  
      - SessionIdType: customKey  
   c. Add Output Parser node (Structured JSON2)  
      - JSON Schema Example:  
        ```json
        {
          "is_query": true,
          "category": "support",
          "reason": "The email asks about shipping availability"
        }
        ```  
   d. Create AI Agent node (Email Classification)  
      - Prompt: Classify email with instructions to return only JSON in above format  
      - Connect nodes in order: OpenAI Chat → Memory → Output Parser → AI Agent

4. **Create If node:**  
   - Condition: `$json.output.is_query == true` (boolean strict equals)  
   - True output: route to GEO Identification block  
   - False output: route to Manual Review Slack notification

5. **Create GEO Identification block:**  
   a. OpenAI Chat Model node (GPT-4o-mini)  
   b. Memory Buffer node (Conversation Buffer)  
      - SessionKey: `"GEO Defined"`  
   c. Output Parser node (Structured JSON)  
      - JSON Schema Example:  
        ```json
        {
          "geo": "Canada"
        }
        ```  
   d. AI Agent node (GEO Identifier)  
      - Prompt instructs AI to return only one GEO label in JSON, no extra text  
   - Connect in sequence: OpenAI Chat → Memory → Output Parser → AI Agent

6. **Create GEO-Optimized Reply block:**  
   a. OpenAI Chat Model node (GPT-4o-mini)  
   b. Memory Buffer node (Conversation Buffer1)  
      - SessionKey: `"GEO Defined"`  
   c. Output Parser node (Structured JSON1)  
      - JSON Schema Example:  
        ```json
        {
          "to": "laura@example.com",
          "subject": "Re: Shipping to Canada?",
          "reply": "Hello Laura, ...",
          "geo_used": "Canada",
          "summary": "Provided shipping info and fees for a customer in Canada."
        }
        ```  
   d. AI Agent node (GEO-Optimized Support)  
      - Prompt defines GEO-specific tone rules and requires strictly formatted JSON reply  
   - Connect: OpenAI Chat → Memory → Output Parser → AI Agent

7. **Create Format Data For Email node (Set node):**  
   - Assign:  
     - `To` = `{{$json.output.to}}`  
     - `Reply` = `{{$json.output.reply}}`  
     - `Geo Used` = `{{$json.output.geo_used}}`  
     - `Summary` = `{{$json.output.summary}}`  
     - `Subject` = `{{$json.output.subject}}`  
   - Input: AI Agent - GEO-Optimized Support output

8. **Create Gmail Send node (Send a message):**  
   - Use formatted fields:  
     - Send To: `{{$json.To}}`  
     - Subject: `{{$json.Subject}}`  
     - Message: `{{$json.Reply}}`  
   - Set Gmail OAuth2 credentials  
   - Email type: text/plain

9. **Create Slack notification node (Notify Slack):**  
   - Channel: select target Slack channel by ID  
   - Message text template:  
     ```
     AI replied to {{$json.To}}
     GEO: {{$json["Geo Used"]}}
     Summary: {{$json.Summary}}
     ```  
   - Set Slack API credentials

10. **Create Manual Review Slack node:**  
    - Channel: same as Notify Slack  
    - Message template:  
      ```
      Manual Review Intervention:- 

      Category:-{{$json.output.category}} 
      Reason{{$json.output.reason}} 

      Please check your Inbox.
      ```  
    - Credentials: Slack API

11. **Create Error Handling block:**  
    a. Add Error Handler Trigger node (triggers on workflow errors)  
    b. Add Slack node (Slack: Send Error Alert)  
       - Message template:  
         ```
         ⚠️ *Error in API Error Catalog Workflow*
         *Node:* {{$json.node.name}}
         *Message:* {{$json.error.message}}
         *Time:* {{$json.timestamp}}
         ```  
       - Credentials: Slack OAuth2 (different account from main Slack)  
    - Connect Error Handler Trigger → Slack Error Alert

12. **Connect all nodes according to logical flow:**  
    - Gmail Trigger → Extract Email Data → AI Agent - Email Classification → If  
    - If (true) → AI Agent - GEO Identifier → AI Agent - GEO-Optimized Support → Format Data For Email → Send a message & Notify Slack  
    - If (false) → Manual Review Slack  
    - Add error handling as separate trigger flow

13. **Validate all JSON schemas in output parsers to ensure AI outputs match exactly.**

14. **Test workflow with sample emails to ensure correct classification, GEO detection, reply generation, sending, and Slack notifications.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                            | Context or Link                                                                                      |
|-----------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| Real-Time GEO-Optimized Answer Engine: Listens for support emails, classifies queries, detects GEO, generates GEO-specific replies, sends via Gmail and notifies Slack. | Workflow overview sticky note                                                                        |
| Strict JSON output format is mandatory for all AI agents to prevent downstream parsing and sending failures.                           | Sticky note warning on strict JSON requirement                                                     |
| GEO tone rules: US (direct, casual), UK (polite, formal), India (warm, guiding), Canada/EU (neutral, compliance-friendly), Australia (friendly, informal) | AI Agent - GEO-Optimized Support node system message                                               |
| Slack notifications cover both successful AI replies and manual review alerts for non-support queries.                                  | Slack notifications sticky note                                                                     |
| Workflow includes robust error handling that reports node errors to Slack for rapid debugging.                                          | Error Handling sticky note and nodes                                                                |
| Setup requires OpenAI, Gmail OAuth2, and Slack API credentials properly configured and authorized.                                      | Setup instructions from overview sticky note                                                       |
| Customization possible by adjusting GEO labels, tone rules, classification criteria, or adding locales.                                | Overview sticky note                                                                                 |

---

**Disclaimer:** The provided text is derived exclusively from an automated workflow created with n8n, a tool for integration and automation. This process strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data processed is legal and publicly accessible.

---