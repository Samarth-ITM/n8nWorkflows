Summarize Calls & Notify Teams via HubSpot, Slack, Email, WhatsApp

https://n8nworkflows.xyz/workflows/summarize-calls---notify-teams-via-hubspot--slack--email--whatsapp-4641


# Summarize Calls & Notify Teams via HubSpot, Slack, Email, WhatsApp

### 1. Workflow Overview

This workflow automates the process of summarizing client call transcripts and notifying relevant departments via multiple communication channels, including email, Slack, WhatsApp, and HubSpot CRM. It is designed for organizations that receive client conversations and need to efficiently categorize feedback, summarize discussions, update CRM records, and notify responsible teams.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Receives the client conversation transcript and metadata via a webhook.
- **1.2 Email Routing Setup:** Defines the responsible email addresses for different departments.
- **1.3 Conversation Summarization:** Uses an AI model to generate a concise summary of the call transcript.
- **1.4 Department Routing Agent:** An AI-powered router decides which department should be notified based on the conversation content, sending emails accordingly.
- **1.5 CRM Update (HubSpot):** Searches for the client in HubSpot by email and updates their record with the summarized meeting notes.
- **1.6 Multi-Channel Notifications:** Sends the conversation summary to Slack and WhatsApp for team awareness.
- **1.7 Email Dispatch:** Sends customized emails to the relevant departments based on routing decisions.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  Receives incoming POST requests containing client conversation transcripts and relevant metadata (e.g., client email, WhatsApp contact number).

- **Nodes Involved:**  
  - Webhook

- **Node Details:**  
  - **Webhook**  
    - Type: HTTP Webhook Listener  
    - Configuration: Listens for POST requests at a specific path (`1ffeacc2-1dbd-4a9e-bf3f-6811ddd31642`).  
    - Inputs: External HTTP POST request with JSON body containing fields like "Client Conversation," "Client Email," "Your Name," and "WhatsApp Contact Number."  
    - Outputs: JSON payload forwarded downstream.  
    - Edge cases: Malformed payloads, missing required fields, webhook timeout or unauthorized requests.  
    - Sticky Note: "Webhook to be used to send the transcription."

#### 2.2 Email Routing Setup

- **Overview:**  
  Defines static email addresses for the departments responsible for product support, invoicing, and commercial/travel issues.

- **Nodes Involved:**  
  - Define routing emails  
  - Sticky Note2

- **Node Details:**  
  - **Define routing emails**  
    - Type: Set Node  
    - Configuration: Assigns fixed strings for three email variables:  
      - Support Email: `info@encoresky.com`  
      - Administrative Email: `hr@encoresky.com`  
      - Commercial Email: `sales@encoresky.com`  
    - Outputs: JSON with these email addresses for use by routing agent.  
    - Edge cases: Emails must be valid; changes require workflow edit.  
  - **Sticky Note2**  
    - Provides context for setting emails for each responsible person.

#### 2.3 Conversation Summarization

- **Overview:**  
  Uses an OpenAI-based summarization chain to produce a concise (2-3 sentence) summary of the client conversation.

- **Nodes Involved:**  
  - Summarization  
  - OpenAI Chat Model

- **Node Details:**  
  - **Summarization**  
    - Type: LangChain Summarization Chain Node  
    - Configuration: Uses the "stuff" method with a prompt instructing to summarize the given conversation text in 2-3 sentences, preserving the original language.  
    - Input: The raw conversation text from the webhook.  
    - Output: Summarized text in `response.text`.  
    - Edge cases: Very long or malformed text could cause API issues; language detection depends on input.  
  - **OpenAI Chat Model**  
    - Type: LangChain OpenAI Chat Model Node  
    - Configuration: Uses GPT-4o-mini model to process language model requests from Summarization and Router Agent nodes.  
    - Credentials: Uses OpenAI API key configured in workflow credentials.  
    - Edge cases: API rate limits, authentication errors, response latency.

#### 2.4 Department Routing Agent

- **Overview:**  
  An AI-powered router analyzes the summarized text and decides which single department should be notified by email, sending a customized email to that department including the client conversation.

- **Nodes Involved:**  
  - Router Agent  
  - Sticky Note1

- **Node Details:**  
  - **Router Agent**  
    - Type: LangChain Agent Node (Router)  
    - Configuration:  
      - Input text is the client conversation from webhook.  
      - System message instructs the agent to identify the relevant department among three options (product, invoicing, travel) and send an email to the corresponding email address defined in "Define routing emails."  
      - Adds the client's email prefixed with "FROM CLIENT:" at the top of the email body.  
      - Uses Mailjet tool (implied integration) to send HTML emails with subject, recipient, and body.  
    - Input connections: From Summarization and Webhook nodes.  
    - Output connections: To Gmail node for sending emails.  
    - Edge cases: Ambiguous conversation topics, multiple relevant departments, email sending failures, API limits.  
  - **Sticky Note1**  
    - Describes the router’s role: analyzing content, deciding relevance, sending emails.

#### 2.5 CRM Update (HubSpot)

- **Overview:**  
  Searches HubSpot CRM for the client by email, then attaches the summarized conversation as meeting notes to the client’s record.

- **Nodes Involved:**  
  - HubSpot Search Client  
  - HubSpot Save Notes  
  - Sticky Note

- **Node Details:**  
  - **HubSpot Search Client**  
    - Type: HubSpot Node (Search operation)  
    - Configuration: Searches contacts by email property matching the client email from webhook.  
    - Credentials: OAuth2 for HubSpot API.  
    - Output: Contact's HubSpot object ID for note association.  
    - Edge cases: Client not found, API authentication errors, rate limits.  
  - **HubSpot Save Notes**  
    - Type: HubSpot Node (Engagement - Meeting)  
    - Configuration: Creates a new meeting note with title "New meeting" and body as the summarized text; associates note with the contact found.  
    - Credentials: OAuth2 HubSpot.  
    - Edge cases: Failed associations, missing contact ID, API errors.  
  - **Sticky Note**  
    - Explains the process of saving transcription to HubSpot.

#### 2.6 Multi-Channel Notifications

- **Overview:**  
  Sends the summarized conversation to Slack and WhatsApp Business Cloud to notify teams in real time.

- **Nodes Involved:**  
  - Slack  
  - WhatsApp Business Cloud  
  - Sticky Note4  
  - Sticky Note3 (context for webhook input)

- **Node Details:**  
  - **Slack**  
    - Type: Slack Node  
    - Configuration: Posts a message containing the latest summary text to a specified Slack channel (`C08VCE4AYNL`). Uses OAuth2 credentials.  
    - Edge cases: Slack API errors, channel permissions, message formatting issues.  
  - **WhatsApp Business Cloud**  
    - Type: WhatsApp Node  
    - Configuration: Sends a templated WhatsApp message (`n8n_workflow_latest|en`) with two parameters: client's name and the summarized conversation. Uses a phone number ID and recipient number from webhook data.  
    - Credentials: WhatsApp Business API OAuth2.  
    - Edge cases: Phone number formatting, template approval, API limits, message delivery failures.  
  - **Sticky Note4**  
    - Notes configuring Slack channel and WhatsApp recipient.  
  - **Sticky Note3**  
    - Notes the webhook is the source of transcription input.

#### 2.7 Email Dispatch

- **Overview:**  
  Sends the customized routing email to the relevant department's address, as determined by the Router Agent.

- **Nodes Involved:**  
  - Gmail

- **Node Details:**  
  - **Gmail**  
    - Type: Gmail Node (Send Email)  
    - Configuration: Sends email using OAuth2 credentials; recipient ("To"), subject, and message body are dynamically set by the Router Agent AI output.  
    - Edge cases: Gmail API quota exceeded, invalid email addresses, authentication failures.  
  - Sticky notes related to email routing are covered under Router Agent block.

---

### 3. Summary Table

| Node Name             | Node Type                          | Functional Role                             | Input Node(s)           | Output Node(s)                | Sticky Note                                                                                                                                |
|-----------------------|----------------------------------|---------------------------------------------|-------------------------|------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------|
| Webhook               | HTTP Webhook                     | Receives client conversation transcription | -                       | Define routing emails         | ## Webhook to be used to send the transcription.                                                                                          |
| Define routing emails  | Set                             | Sets email addresses for departments        | Webhook                  | Summarization                 | ## Set emails here for each responsible person                                                                                            |
| Summarization         | LangChain Summarization Chain    | Summarizes client conversation               | Define routing emails     | Router Agent, HubSpot Search Client, Slack | The transcript summarized here                                                                                                            |
| OpenAI Chat Model      | LangChain OpenAI Chat Model      | Provides AI language model for summarization and routing | Summarization, Router Agent (ai_languageModel) | Summarization, Router Agent    |                                                                                                                                            |
| Router Agent          | LangChain Agent (Router)          | Decides relevant department and composes email | Summarization             | Gmail                        | ## Router agent: Analyzes content, decides relevance, sends emails to departments                                                         |
| Gmail                 | Gmail Tool                      | Sends email to the relevant department      | Router Agent              | -                            |                                                                                                                                            |
| HubSpot Search Client  | HubSpot Node (Search)            | Finds client record in HubSpot by email     | Summarization             | HubSpot Save Notes            | Search for the client ID based on email                                                                                                   |
| HubSpot Save Notes     | HubSpot Node (Meeting Engagement) | Adds summarized notes to client record       | HubSpot Search Client     | -                            | Save transcription to HubSpot: Add meeting notes                                                                                          |
| Slack                 | Slack Node                      | Posts summarized conversation to Slack      | Summarization             | WhatsApp Business Cloud       | ## Send the summary to Slack and WhatsApp: Configure Slack Channel and WhatsApp Recipient                                                  |
| WhatsApp Business Cloud| WhatsApp Node                   | Sends templated WhatsApp message             | Slack                     | -                            |                                                                                                                                            |
| Sticky Note1          | Sticky Note                     | Explains Router Agent logic                   | -                        | -                            | ## Router agent: Makes decisions with help of LLM, analyzes content, informs departments                                                   |
| Sticky Note2          | Sticky Note                     | Notes on setting emails                       | -                        | -                            | ## Set emails here for each responsible person                                                                                            |
| Sticky Note3          | Sticky Note                     | Notes about webhook input                     | -                        | -                            | ## Webhook to be used to send the transcription.                                                                                          |
| Sticky Note4          | Sticky Note                     | Notes on Slack and WhatsApp notification setup | -                        | -                            | ## Send the summary to Slack and WhatsApp: Configure Slack Channel and WhatsApp Recipient                                                  |
| Sticky Note           | Sticky Note                     | Notes about saving transcription to HubSpot | -                        | -                            | ## Save transcription to HubSpot: Search for client ID, upload summarized conversation as meeting notes                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Type: Webhook  
   - Path: `1ffeacc2-1dbd-4a9e-bf3f-6811ddd31642`  
   - HTTP Method: POST  
   - Purpose: Receive JSON payload with fields "Client Conversation," "Client Email," "Your Name," "WhatsApp Contact Number."

2. **Create Set Node (Define routing emails)**  
   - Assign variables:  
     - `Support Email` = `info@encoresky.com`  
     - `Administrative Email` = `hr@encoresky.com`  
     - `Commercial Email` = `sales@encoresky.com`  
   - Connect Webhook main output to this node.

3. **Create LangChain Summarization Node**  
   - Use the "stuff" summarization method.  
   - Prompt:  
     ```
     Summarize the following Conversation in 2-3 sentences:

     "{{ $json['Client Conversation'] }}"

     Just output the summarized conversation and nothing else. Use the same language as the input.
     ```  
   - Input: Connect from "Define routing emails" node.  
   - Version: Use latest compatible LangChain Summarization node.

4. **Create LangChain OpenAI Chat Model Node**  
   - Model: `gpt-4o-mini`  
   - Credentials: Configure with OpenAI API key (OpenAiApi credentials).  
   - Connect as AI language model provider to Summarization and Router Agent nodes.

5. **Create LangChain Router Agent Node**  
   - Input text: `{{ $json.body['Client Conversation'] }}` from Webhook node.  
   - System message:  
     ```
     You are provided with some client-company conversation and should decide who has to be informed about the feedback. Always only inform one person. Those are your options: 
     - It's about a product, send an email to {{ $json['Support Email'] }}
     - It's about an invoicing problem, send an email to {{ $json['Administrative Email'] }}
     - It's related to a travelling/trip, send an email to {{ $json['Commercial Email'] }}

     Add the email of the person ("{{ $json.body['Client Email'] }}") at the beginning of the text preceded by "FROM CLIENT: "
     Use the Mailjet tool to inform each of the most related department. Provide mailjet with a subject, an email, and the email body formatted as html which is the client conversation itself.
     ```  
   - Connect Summarization output and Webhook input to this node.  
   - Connect OpenAI Chat Model node as AI language model provider.

6. **Create Gmail Node**  
   - Operation: Send Email  
   - Parameters:  
     - To: Dynamic input from Router Agent AI output  
     - Subject: Dynamic input from Router Agent AI output  
     - Message: Dynamic input from Router Agent AI output  
     - Disable attribution append  
   - Credentials: Gmail OAuth2 configured.  
   - Connect Router Agent output to Gmail input.

7. **Create HubSpot Search Client Node**  
   - Operation: Search Contacts  
   - Filter: Email equals `{{ $json.body['Client Email'] }}` from Webhook.  
   - Credentials: HubSpot OAuth2 configured.  
   - Connect Summarization output to this node.

8. **Create HubSpot Save Notes Node**  
   - Operation: Create Engagement (Meeting)  
   - Title: "New meeting"  
   - Body: Summarization output text  
   - Association: Contact ID from "HubSpot Search Client" output  
   - Credentials: HubSpot OAuth2 configured.  
   - Connect HubSpot Search Client output to this node.

9. **Create Slack Node**  
   - Operation: Post Message  
   - Channel: `C08VCE4AYNL` (or your Slack Channel ID)  
   - Message Text: `"Below is the latest summary.\n\n{{ $json.response.text }}"` from Summarization node.  
   - Authentication: OAuth2 Slack credentials.  
   - Connect Summarization output to Slack node.

10. **Create WhatsApp Business Cloud Node**  
    - Template Name: `n8n_workflow_latest|en`  
    - Components: Use body parameters with two text parameters:  
      - `{{ $json.body['Your Name'] }}` from Webhook  
      - `{{ $json.response.text }}` from Summarization  
    - Phone Number ID: Your WhatsApp Business phone number ID  
    - Recipient Phone Number: `{{ $json.body['WhatsApp Contact Number'] }}` from Webhook  
    - Credentials: WhatsApp Business API OAuth2 configured.  
    - Connect Slack node output to WhatsApp node.

11. **Add Sticky Notes** at appropriate positions for documentation and clarity as described in the original workflow.

12. **Set execution order** as per dependencies: Webhook → Define routing emails → Summarization → Router Agent (and HubSpot Search, Slack) → Gmail and HubSpot Save Notes → WhatsApp.

---

### 5. General Notes & Resources

| Note Content                                                                                                                              | Context or Link                                          |
|-------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------|
| The Router Agent uses a Mailjet tool internally for sending emails to departments. Mailjet credentials/configuration are assumed.         | Router Agent node description                             |
| Slack channel ID `C08VCE4AYNL` should be replaced with your actual Slack channel ID if deploying in a different workspace.                 | Slack Node parameter                                     |
| WhatsApp message template `n8n_workflow_latest|en` must be pre-approved in WhatsApp Business Cloud API for successful message sending.    | WhatsApp Business Cloud Node                             |
| OpenAI API key must have access to GPT-4o-mini or equivalent model.                                                                       | LangChain OpenAI Chat Model node                         |
| HubSpot OAuth2 must have permissions to search contacts and create meeting engagements.                                                   | HubSpot Nodes documentation                              |
| The workflow assumes the client email and WhatsApp contact number are provided in webhook payload; validate presence before triggering. | Input data requirements                                  |

---

**Disclaimer:**  
The text provided originates exclusively from an automated workflow created with n8n, a tool for integration and automation. This processing strictly complies with content policies and contains no illegal, offensive, or protected material. All data processed is legal and publicly available.