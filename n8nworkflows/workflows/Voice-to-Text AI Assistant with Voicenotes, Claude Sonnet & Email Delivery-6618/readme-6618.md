Voice-to-Text AI Assistant with Voicenotes, Claude Sonnet & Email Delivery

https://n8nworkflows.xyz/workflows/voice-to-text-ai-assistant-with-voicenotes--claude-sonnet---email-delivery-6618


# Voice-to-Text AI Assistant with Voicenotes, Claude Sonnet & Email Delivery

### 1. Workflow Overview

This workflow, titled **"Voice-to-Text AI Assistant with Voicenotes, Claude Sonnet & Email Delivery"**, automates the process of receiving voice notes transcribed into text, enhancing and processing the text via an AI agent, storing the input and AI-generated output in a database, and finally sending a formatted email with the AI response. It is designed for use cases where voicenotes or audio transcripts need to be intelligently analyzed, summarized, or expanded into detailed structured documents and then shared via email.

**Logical blocks:**

- **1.1 Input Reception & Data Preparation:** Reception of webhook calls carrying transcription data, extraction and normalization of relevant fields.
- **1.2 AI Processing:** Feeding the cleaned prompt into an AI agent (LangChain with Claude Sonnet model) to generate a detailed structured response.
- **1.3 Data Enrichment & Storage:** Preparing and saving both the prompt and AI output into NocoDB tables for record-keeping.
- **1.4 Output Formatting & Email Delivery:** Converting AI output from markdown to HTML, embedding it in a styled email template, and sending it through Gmail.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Data Preparation

- **Overview:**  
  This block receives incoming HTTP POST requests from Voicenotes (or similar services), extracts and cleans relevant data fields such as the transcription prompt, title, and timestamp, making them ready for AI processing.

- **Nodes Involved:**  
  - Webhook  
  - Edit Fields  
  - Sticky Note2 (comment)  
  - Sticky Note (comment)

- **Node Details:**

  - **Webhook**  
    - *Type:* `n8n-nodes-base.webhook`  
    - *Role:* Receives POST requests from external sources (Voicenotes).  
    - *Configuration:* Listens on path `dbedd9a0-e0bd-4e70-8f62-87013f9dd9c3` using HTTP POST.  
    - *Input/Output:* Input externally triggered; outputs raw JSON payload.  
    - *Edge cases:* Missing or malformed POST data, unauthorized requests (no explicit auth configured).  
    - *Sticky Notes:* "New note with 'prompts' tag is delivered by Voicenotes by webhook"  

  - **Edit Fields**  
    - *Type:* `n8n-nodes-base.set`  
    - *Role:* Normalize and map incoming JSON fields to simplified names for downstream use.  
    - *Configuration:*  
      - `created` set to the incoming `body.timestamp`  
      - `note_title` to `body.data.title`  
      - `prompt` to `body.data.transcript`  
    - *Expressions:* Uses expressions like `={{ $json.body.timestamp }}` to extract nested fields.  
    - *Input:* Output of Webhook  
    - *Output:* JSON with simplified keys `created`, `note_title`, `prompt`  
    - *Edge cases:* Missing nested fields may cause empty outputs or errors in expressions.  
    - *Sticky Notes:* "Set the fields to remove CF headers (etc)"  

---

#### 2.2 AI Processing

- **Overview:**  
  This block submits the extracted prompt to an AI agent powered by LangChain, specifically using the Claude Sonnet 4 model via OpenRouter, to generate a detailed, well-structured textual response according to a strict system message guidance.

- **Nodes Involved:**  
  - AI Agent  
  - OpenRouter Chat Model  
  - Edit Fields1  
  - Sticky Note1 (comment)

- **Node Details:**

  - **AI Agent**  
    - *Type:* `@n8n/n8n-nodes-langchain.agent`  
    - *Role:* Sends prompt text to the AI model, receives structured textual output.  
    - *Configuration:*  
      - `text` input is dynamically set as `={{ $json.prompt }}` from previous node.  
      - System message instructs the AI to generate detailed, well-structured documents with headers, summary tables, and source URLs, tailored for a user named Daniel.  
      - `promptType` set to `"define"`.  
    - *Input:* From `Edit Fields` output.  
    - *Output:* AI-generated response stored in `output` field.  
    - *Edge cases:* AI API failures, timeouts, prompt length limits, or malformed prompt input.  
    - *Sub-workflow:* Uses LangChain integration with OpenRouter Chat Model as the language model backend.  
    - *Sticky Notes:* "Prompt is run into AI agent"  

  - **OpenRouter Chat Model**  
    - *Type:* `@n8n/n8n-nodes-langchain.lmChatOpenRouter`  
    - *Role:* Provides AI language model interface for the AI Agent node.  
    - *Configuration:* Uses model `"anthropic/claude-sonnet-4"` with OpenRouter API credentials.  
    - *Input:* Connected as AI language model for AI Agent node.  
    - *Edge cases:* Auth failures, API rate limits, or model unavailability.  

  - **Edit Fields1**  
    - *Type:* `n8n-nodes-base.set`  
    - *Role:* Collect and prepare fields for database storage after AI processing.  
    - *Configuration:*  
      - Copies `note_title` and `prompt` from `Edit Fields`.  
      - Stores AI `output` from `AI Agent`.  
      - Copies `created` timestamp from `Webhook`.  
    - *Edge cases:* Missing data if previous nodes failed.  

---

#### 2.3 Data Enrichment & Storage

- **Overview:**  
  This block saves both the original prompt and the AI-generated output into two separate tables in NocoDB for persistent storage and future retrieval.

- **Nodes Involved:**  
  - Save Prompt  
  - Save Output  
  - Sticky Note3 (comment)

- **Node Details:**

  - **Save Prompt**  
    - *Type:* `n8n-nodes-base.nocoDb`  
    - *Role:* Creates a new record in the NocoDB table for prompts.  
    - *Configuration:*  
      - Table: `mg36i2thdctcg5b` (prompt storage)  
      - Fields: stores `c76za9u4d0lhiyx` with the prompt string.  
      - Auth: Uses NocoDB API token credentials.  
    - *Input:* From `Edit Fields1` node.  
    - *Output:* Confirmation of record creation.  
    - *Edge cases:* API token invalid, table or field name changes, network issues.  

  - **Save Output**  
    - *Type:* `n8n-nodes-base.nocoDb`  
    - *Role:* Creates a new record in NocoDB for AI-generated outputs.  
    - *Configuration:*  
      - Table: `mn2ttgi7atggy1t` (output storage)  
      - Fields: AI output text, original prompt, and a static source label "Voicenotes.com Automation (N8N)".  
      - Auth: Same NocoDB API token credentials.  
    - *Input:* From `Save Prompt` node output.  
    - *Edge cases:* Same as Save Prompt node.  

  - *Sticky Notes:* "Chain 1: the note is saved to NocoDB"  

---

#### 2.4 Output Formatting & Email Delivery

- **Overview:**  
  Converts the AI-generated markdown response into HTML, prepares a well-styled HTML email with prompt and response sections, and sends it via Gmail SMTP with OAuth2.

- **Nodes Involved:**  
  - Markdown  
  - Edit Fields2  
  - Send a message  
  - Sticky Note4 (comment)

- **Node Details:**

  - **Markdown**  
    - *Type:* `n8n-nodes-base.markdown`  
    - *Role:* Converts AI output from markdown syntax to HTML for email compatibility.  
    - *Configuration:* Mode set to `markdownToHtml`.  
    - *Input:* From `AI Agent` output.  
    - *Output:* HTML string of AI response.  
    - *Edge cases:* Large markdown may cause processing delays or truncation.  

  - **Edit Fields2**  
    - *Type:* `n8n-nodes-base.set`  
    - *Role:* Prepare final email data fields for sending.  
    - *Configuration:*  
      - `title` from original webhook title  
      - `prompt` from webhook transcript  
      - `output` from Markdown HTML data  
    - *Input:* From `Markdown`.  
    - *Edge cases:* Missing or malformed HTML input.  

  - **Send a message**  
    - *Type:* `n8n-nodes-base.gmail`  
    - *Role:* Sends the formatted AI response email to a fixed recipient.  
    - *Configuration:*  
      - Recipient: `ai-outputs@yourdomain.com` (static)  
      - Subject: Set dynamically to the note title with suffix `(Voicenotes prompt)`  
      - Message: Full HTML email composed with inline CSS styling, including sections for Title, Prompt, and AI Response with distinct background colors and formatting.  
      - Sender name: "DSR Holdings Apps"  
      - Credentials: Gmail OAuth2 with stored token.  
    - *Edge cases:* OAuth token expiration, Gmail API quota limits, invalid email addresses, large email payloads.  
    - *Sticky Notes:* "Chain 2: the note is converted into HTML and then injected into an email template"  

---

### 3. Summary Table

| Node Name           | Node Type                              | Functional Role                              | Input Node(s)      | Output Node(s)       | Sticky Note                                                                                                   |
|---------------------|--------------------------------------|----------------------------------------------|--------------------|----------------------|---------------------------------------------------------------------------------------------------------------|
| Webhook             | n8n-nodes-base.webhook                | Receives incoming Voicenotes POST data       | External trigger   | Edit Fields          | New note with 'prompts' tag is delivered by Voicenotes by webhook                                             |
| Edit Fields         | n8n-nodes-base.set                   | Normalize and extract relevant fields        | Webhook            | AI Agent             | Set the fields to remove CF headers (etc)                                                                     |
| AI Agent            | @n8n/n8n-nodes-langchain.agent       | Sends prompt to AI and receives response     | Edit Fields        | Edit Fields1, Markdown| Prompt is run into AI agent                                                                                   |
| OpenRouter Chat Model| @n8n/n8n-nodes-langchain.lmChatOpenRouter | Provides AI model backend for AI Agent       | - (linked)         | AI Agent             |                                                                                                               |
| Edit Fields1        | n8n-nodes-base.set                   | Prepare data for database storage             | AI Agent           | Save Prompt           |                                                                                                               |
| Save Prompt          | n8n-nodes-base.nocoDb                | Stores prompt in NocoDB table                  | Edit Fields1       | Save Output           | Chain 1: the note is saved to NocoDB                                                                           |
| Save Output          | n8n-nodes-base.nocoDb                | Stores AI output in NocoDB table               | Save Prompt        | (none)                |                                                                                                               |
| Markdown             | n8n-nodes-base.markdown              | Convert AI markdown output to HTML             | AI Agent           | Edit Fields2          |                                                                                                               |
| Edit Fields2         | n8n-nodes-base.set                   | Prepare email fields (title, prompt, output)  | Markdown           | Send a message        |                                                                                                               |
| Send a message       | n8n-nodes-base.gmail                 | Send formatted AI response email               | Edit Fields2       | (none)                | Chain 2: the note is converted into HTML and then injected into an email template                              |
| Sticky Note2         | n8n-nodes-base.stickyNote            | Comment                                        | -                  | -                     | New note with 'prompts' tag is delivered by Voicenotes by webhook                                             |
| Sticky Note          | n8n-nodes-base.stickyNote            | Comment                                        | -                  | -                     | Set the fields to remove CF headers (etc)                                                                     |
| Sticky Note1         | n8n-nodes-base.stickyNote            | Comment                                        | -                  | -                     | Prompt is run into AI agent                                                                                   |
| Sticky Note3         | n8n-nodes-base.stickyNote            | Comment                                        | -                  | -                     | Chain 1: the note is saved to NocoDB                                                                           |
| Sticky Note4         | n8n-nodes-base.stickyNote            | Comment                                        | -                  | -                     | Chain 2: the note is converted into HTML and then injected into an email template                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Type: `Webhook`  
   - HTTP Method: `POST`  
   - Path: `dbedd9a0-e0bd-4e70-8f62-87013f9dd9c3`  
   - Purpose: Receive incoming transcription payloads.  

2. **Create Edit Fields Node (Edit Fields)**  
   - Type: `Set`  
   - Assign fields:  
     - `created` = `={{ $json.body.timestamp }}`  
     - `note_title` = `={{ $json.body.data.title }}`  
     - `prompt` = `={{ $json.body.data.transcript }}`  
   - Connect input: `Webhook` → `Edit Fields`  

3. **Create AI Agent Node**  
   - Type: `LangChain Agent`  
   - Text Input: `={{ $json.prompt }}` (from previous node)  
   - System Message:  
     ```
     Your task is to act as a helpful assistant to the user change-to-your-name. You'll be receiving prompts through a workflow, so should respond with a detailed and thorough answer adhering to a single turn workflow. Daniel likes to receive well structured documents with headers to delineate between sections and when comparing products or information likes to receive a summary table at the end of the document providing a comparison matrix as well as a executive summary section at the start of the document Deliver thorough responses according to these principles, ensure accurate information and provide URLs referenced at the end of the documents in a separate sources section.
     ```  
   - Prompt Type: `define`  
   - Connect input: `Edit Fields` → `AI Agent`  

4. **Create OpenRouter Chat Model Node**  
   - Type: `LangChain OpenRouter Chat Model`  
   - Model: `anthropic/claude-sonnet-4`  
   - Credentials: OpenRouter API (set with your API key)  
   - Connect as AI language model for `AI Agent` node.  

5. **Create Edit Fields1 Node**  
   - Type: `Set`  
   - Assign fields:  
     - `note_title` = `={{ $('Edit Fields').item.json.note_title }}`  
     - `prompt` = `={{ $('Edit Fields').item.json.prompt }}`  
     - `output` = `={{ $json.output }}` (from AI Agent)  
     - `created` = `={{ $('Webhook').item.json.body.timestamp }}`  
   - Connect input: `AI Agent` → `Edit Fields1`  

6. **Create Save Prompt Node**  
   - Type: `NocoDB`  
   - Operation: `Create`  
   - Table: `mg36i2thdctcg5b` (prompts table)  
   - Fields: map prompt text to appropriate field (e.g., `c76za9u4d0lhiyx`) with `={{ $json.prompt }}`  
   - Credentials: NocoDB API token  
   - Connect input: `Edit Fields1` → `Save Prompt`  

7. **Create Save Output Node**  
   - Type: `NocoDB`  
   - Operation: `Create`  
   - Table: `mn2ttgi7atggy1t` (outputs table)  
   - Fields:  
     - AI Output field with `={{ $('AI Agent').item.json.output }}`  
     - Prompt field with `={{ $('Edit Fields1').item.json.prompt }}`  
     - Source field set to static `"Voicenotes.com Automation (N8N)"`  
   - Credentials: NocoDB API token  
   - Connect input: `Save Prompt` → `Save Output`  

8. **Create Markdown Node**  
   - Type: `Markdown`  
   - Mode: `markdownToHtml`  
   - Input: AI Agent's output markdown text  
   - Connect input: `AI Agent` → `Markdown`  

9. **Create Edit Fields2 Node**  
   - Type: `Set`  
   - Assign fields:  
     - `title` = `={{ $('Webhook').item.json.body.data.title }}`  
     - `prompt` = `={{ $('Webhook').item.json.body.data.transcript }}`  
     - `output` = `={{ $json.data }}` (HTML from Markdown node)  
   - Connect input: `Markdown` → `Edit Fields2`  

10. **Create Send a message Node**  
    - Type: `Gmail`  
    - Recipient Email: `ai-outputs@yourdomain.com` (replace with actual)  
    - Subject: `={{ $json.title }} (Voicenotes prompt)`  
    - Message (HTML): Copy the full HTML template from the workflow, including inline CSS and placeholders for `{{ $json.title }}`, `{{ $json.prompt }}`, and `{{ $json.output }}`  
    - Sender Name: `"DSR Holdings Apps"`  
    - Credentials: Gmail OAuth2 (configure with your Google account and OAuth credentials)  
    - Connect input: `Edit Fields2` → `Send a message`  

11. **Add Sticky Notes (Optional)**  
    - Add sticky notes with the comments as in the original flow for documentation and clarity.  

12. **Set Execution Order and Activate**  
    - Confirm all connections and set execution order to 'v1' (default)  
    - Activate workflow to listen for webhooks and process incoming voice note prompts automatically.

---

### 5. General Notes & Resources

| Note Content                                                                                                             | Context or Link                                           |
|--------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------|
| Workflow demo and explanation related to Voicenotes and AI processing with LangChain and Claude Sonnet models.           | N/A                                                       |
| Usage of NocoDB as a no-code backend for storing prompts and AI outputs enables easy data management and retrieval.       | [NocoDB Documentation](https://docs.nocodb.com/)          |
| Gmail node uses OAuth2 authentication; ensure token is refreshed and permissions include sending emails on behalf of user.| [n8n Gmail Node OAuth2 Setup](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.gmail/) |
| AI Agent node uses LangChain integration; system message is crucial to control the style and structure of AI responses.  | [LangChain n8n Integration](https://n8n.io/integrations/langchain) |
| Inline CSS in email ensures consistent styling across major email clients; tested best practices are applied.            | [Email Design Guidelines](https://www.campaignmonitor.com/email-design-guide/) |

---

This document fully describes the workflow's architecture, logic, node configurations, potential failure points, and instructions for fully reproducing the setup in n8n without referring to the raw JSON.