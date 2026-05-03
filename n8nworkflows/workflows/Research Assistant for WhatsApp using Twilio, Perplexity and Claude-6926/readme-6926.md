Research Assistant for WhatsApp using Twilio, Perplexity and Claude

https://n8nworkflows.xyz/workflows/research-assistant-for-whatsapp-using-twilio--perplexity-and-claude-6926


# Research Assistant for WhatsApp using Twilio, Perplexity and Claude

---
### 1. Workflow Overview

This n8n workflow, titled **"Research Assistant for WhatsApp using Twilio, Perplexity and Claude"**, is designed to provide an automated, AI-powered research assistant accessible through WhatsApp. When a user sends a query via WhatsApp, the workflow performs a deep, multi-source research on the topic using Perplexity’s Sonar Pro AI model, refines the output for WhatsApp readability and style with a Claude language model, and then sends the polished summary back to the user via Twilio.

The workflow is logically divided into four main functional blocks:

- **1.1 WhatsApp Input Reception:** Captures incoming WhatsApp messages through a Twilio webhook.
- **1.2 AI Research Processing:** Performs a deep research query using Perplexity’s Sonar Pro model to generate a structured Markdown report.
- **1.3 Content Refinement for WhatsApp:** Refines and formats the raw AI-generated report into a concise, emoji-enhanced message tailored for WhatsApp’s character limits using Claude.
- **1.4 WhatsApp Response Dispatch:** Sends the polished research summary back to the user via Twilio’s WhatsApp messaging API.

---

### 2. Block-by-Block Analysis

#### 2.1 WhatsApp Input Reception

- **Overview:**  
  This block acts as the entry point, receiving incoming messages sent to a Twilio WhatsApp number via a webhook POST request. It extracts the user’s query, sender, and recipient information for downstream processing.

- **Nodes Involved:**  
  - Fetch Whatsapp Request  
  - Sticky Note (📲 WhatsApp Gateway)

- **Node Details:**

  - **Fetch Whatsapp Request**  
    - **Type:** Webhook (n8n-nodes-base.webhook)  
    - **Role:** Receives POST requests from Twilio when a WhatsApp message arrives.  
    - **Configuration:**  
      - HTTP Method: POST  
      - Path: `fetch-whatsapp-request`  
    - **Key Variables:**  
      - Extracts `body.Body` (user’s message text)  
      - Extracts `body.From` (user’s WhatsApp number)  
      - Extracts `body.To` (bot’s WhatsApp number)  
    - **Connections:** Output connects to `Perform Deep Research` node.  
    - **Edge Cases:**  
      - Malformed or empty message bodies may cause empty queries downstream.  
      - Webhook URL must be public and correctly configured in Twilio.  
      - Authentication or request validation is not explicitly shown; potential security considerations.  
    - **Sticky Note:** Describes this node as the front door capturing Twilio WhatsApp messages and passing user input forward.

---

#### 2.2 AI Research Processing

- **Overview:**  
  This block performs the core research task by sending the user’s query to Perplexity’s Sonar Pro AI model. It generates a detailed, well-structured Markdown research report based on multiple sources with citations.

- **Nodes Involved:**  
  - Perform Deep Research (Perplexity)  
  - Sticky Note (🧠 The AI Research Engine)

- **Node Details:**

  - **Perform Deep Research**  
    - **Type:** Perplexity AI node (n8n-nodes-base.perplexity)  
    - **Role:** Executes advanced research using the `sonar-pro` model.  
    - **Configuration:**  
      - Model: sonar-pro  
      - Message Content: A prompt instructing the AI to produce a Markdown-formatted research report with a strict structure including Title, Overview, Key Points, Detailed Analysis, Conclusion, and Sources.  
      - Input Query: Dynamically injected from `{{ $json.body.Body }}` (user’s WhatsApp message).  
      - Simplify: true (likely to streamline output)  
    - **Credentials:** Uses Perplexity API credentials.  
    - **Connections:** Output feeds into `Refine Generated Output for Whatsapp`.  
    - **Edge Cases:**  
      - API key/authentication failures.  
      - Timeout or rate limits from Perplexity API.  
      - Unexpected or malformed user queries may cause incomplete or irrelevant results.  
      - Output may exceed WhatsApp message length limits, requiring downstream truncation.  
    - **Sticky Note:** Explains this node as the core researcher producing a raw Markdown report.

---

#### 2.3 Content Refinement for WhatsApp

- **Overview:**  
  This block refines and formats the raw Markdown report from Perplexity into a concise, engaging WhatsApp-friendly message. It applies stylistic edits, adds emojis, enforces WhatsApp-specific formatting rules, and ensures the total message length respects the 1600-character limit.

- **Nodes Involved:**  
  - Refine Generated Output for Whatsapp (LangChain Claude)  
  - Claude Sonnet 4 (LangChain Claude Model)  
  - Sticky Note (🎨 The Content Stylist)

- **Node Details:**

  - **Refine Generated Output for Whatsapp**  
    - **Type:** LangChain Chain LLM (Chat) node (`@n8n/n8n-nodes-langchain.chainLlm`)  
    - **Role:** Processes the raw research report text through a Claude language model prompt to produce a polished WhatsApp message.  
    - **Configuration:**  
      - Prompt: Detailed instructions to preserve core content, add emojis to title and headers, apply single asterisk bold formatting, use hyphen bullet points, double line breaks between sections, preserve inline citations, and respect 1600-character limit.  
      - Input Text: Injected from `{{ $json.message }}` (output from Perplexity node).  
      - Prompt Type: Define (custom prompt).  
    - **Connections:** Output goes to `Claude Sonnet 4` for final LLM processing, then to `Send Output in Whatsapp`.  
    - **Edge Cases:**  
      - LLM may produce output exceeding character limit if not properly constrained.  
      - Formatting errors if the input Markdown is malformed.  
      - Possible API errors or timeouts from LangChain or Anthropic.  
    - **Sub-Workflow:** This node internally calls the Claude Sonnet 4 node for language model refinement.  

  - **Claude Sonnet 4**  
    - **Type:** LangChain LLM Chat node (`@n8n/n8n-nodes-langchain.lmChatAnthropic`)  
    - **Role:** Executes the Claude model (`claude-sonnet-4-20250514`) to generate the final polished text.  
    - **Configuration:**  
      - Model: Claude Sonnet 4  
    - **Credentials:** Uses Anthropic API credentials.  
    - **Connections:** Feeds output back into `Refine Generated Output for Whatsapp` node chain.  
    - **Edge Cases:**  
      - API errors, rate limits, or authentication failures.  
      - Model response latency.  

---

#### 2.4 WhatsApp Response Dispatch

- **Overview:**  
  This final block sends the polished research summary back to the user on WhatsApp using Twilio’s messaging API. It dynamically swaps the sender and recipient numbers to reply correctly.

- **Nodes Involved:**  
  - Send Output in Whatsapp (Twilio)  
  - Sticky Note (🚀 WhatsApp Dispatch)

- **Node Details:**

  - **Send Output in Whatsapp**  
    - **Type:** Twilio Node (n8n-nodes-base.twilio)  
    - **Role:** Sends an outbound WhatsApp message via Twilio’s API.  
    - **Configuration:**  
      - To: Extracted from `$('Fetch Whatsapp Request').item.json.body.From` (user’s WhatsApp number), stripped of `whatsapp:` prefix.  
      - From: Extracted from `$('Fetch Whatsapp Request').item.json.body.To` (bot’s WhatsApp number), stripped of `whatsapp:` prefix.  
      - Message: Uses the polished text from `{{ $json.text }}` (output of content refinement).  
      - Options: Default.  
      - ToWhatsapp: true (ensures WhatsApp message type).  
    - **Credentials:** Uses Twilio API credentials with WhatsApp enabled.  
    - **Edge Cases:**  
      - Twilio API authentication or quota limits.  
      - Invalid phone number formatting can cause message failures.  
      - Network or API latency issues.  
    - **Sticky Note:** Notes the automatic sender/recipient swap and final delivery of the polished message.

---

### 3. Summary Table

| Node Name                      | Node Type                              | Functional Role                       | Input Node(s)               | Output Node(s)                       | Sticky Note                                                                                                                                                                                                                 |
|-------------------------------|--------------------------------------|------------------------------------|-----------------------------|------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Fetch Whatsapp Request         | Webhook (n8n-nodes-base.webhook)     | Receives incoming WhatsApp messages | None                        | Perform Deep Research               | 📲 WhatsApp Gateway: Front door capturing Twilio WhatsApp messages and passing user input forward.                                                                                                                            |
| Perform Deep Research          | Perplexity AI (n8n-nodes-base.perplexity) | Performs deep AI research on query  | Fetch Whatsapp Request       | Refine Generated Output for Whatsapp | 🧠 The AI Research Engine: Core researcher producing raw Markdown report with citations.                                                                                                                                     |
| Refine Generated Output for Whatsapp | LangChain Chain LLM (chat)            | Polishes and formats report for WhatsApp | Perform Deep Research        | Send Output in Whatsapp, Claude Sonnet 4 | 🎨 The Content Stylist: Edits report, adds emojis, enforces formatting, and fits 1600-char limit for WhatsApp.                                                                                                                 |
| Claude Sonnet 4               | LangChain LLM Chat (Anthropic)       | Runs Claude model for text polishing | Refine Generated Output for Whatsapp | Refine Generated Output for Whatsapp |                                                                                                                                                                                                                              |
| Send Output in Whatsapp        | Twilio (n8n-nodes-base.twilio)       | Sends polished message back via WhatsApp | Refine Generated Output for Whatsapp | None                               | 🚀 WhatsApp Dispatch: Sends final formatted text back to user, swapping From/To numbers automatically.                                                                                                                      |
| Sticky Note                   | Sticky Note                          | Documentation/comment only          | None                        | None                               | See sticky notes listed in node details for contextual insights.                                                                                                                                                             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node: "Fetch Whatsapp Request"**  
   - Type: Webhook (n8n-nodes-base.webhook)  
   - HTTP Method: POST  
   - Path: `fetch-whatsapp-request`  
   - Purpose: Receive incoming WhatsApp messages from Twilio.  
   - Output: Passes JSON with `body.Body`, `body.From`, `body.To` to next node.  

2. **Create Perplexity Node: "Perform Deep Research"**  
   - Type: Perplexity AI (n8n-nodes-base.perplexity)  
   - Model: `sonar-pro`  
   - Message: Set prompt to instruct AI to produce a structured Markdown research report. Include these key parts: Title, Overview, Key Points, Detailed Analysis, Conclusion, and Sources.  
   - Inject user query via expression: `{{ $json.body.Body }}`  
   - Simplify: true  
   - Credentials: Configure Perplexity API credentials.  
   - Connect input from `Fetch Whatsapp Request`.  

3. **Create LangChain Chain LLM Node: "Refine Generated Output for Whatsapp"**  
   - Type: LangChain Chain LLM (chat) (`@n8n/n8n-nodes-langchain.chainLlm`)  
   - Prompt: Provide detailed instructions to transform raw Markdown into a WhatsApp-ready message:  
     - Preserve facts, add emojis, single-asterisk bold, hyphen bullets, double line breaks, preserve citations, limit to 1600 chars, no preamble.  
   - Input text from `{{ $json.message }}` (output of Perplexity).  
   - Connect input from `Perform Deep Research`.  

4. **Create LangChain LLM Chat Anthropic Node: "Claude Sonnet 4"**  
   - Type: LangChain LLM Chat Anthropic (`@n8n/n8n-nodes-langchain.lmChatAnthropic`)  
   - Model: `claude-sonnet-4-20250514`  
   - Credentials: Configure Anthropic API credentials.  
   - Connect input from `Refine Generated Output for Whatsapp` (ai_languageModel input).  
   - Connect output back to `Refine Generated Output for Whatsapp` (main output).  

5. **Create Twilio Node: "Send Output in Whatsapp"**  
   - Type: Twilio (n8n-nodes-base.twilio)  
   - To: Expression `={{ $('Fetch Whatsapp Request').item.json.body.From.replace('whatsapp:', '') }}`  
   - From: Expression `={{ $('Fetch Whatsapp Request').item.json.body.To.replace('whatsapp:', '') }}`  
   - Message: Expression `={{ $json.text }}` (output from Refine Generated Output for Whatsapp)  
   - ToWhatsapp: true  
   - Credentials: Configure Twilio API credentials with WhatsApp messaging enabled.  
   - Connect input from `Refine Generated Output for Whatsapp`.  

6. **Connect all nodes in this order:**  
   `Fetch Whatsapp Request` -> `Perform Deep Research` -> `Refine Generated Output for Whatsapp` -> `Send Output in Whatsapp`  
   Also connect `Refine Generated Output for Whatsapp` -> `Claude Sonnet 4` (ai_languageModel input) and `Claude Sonnet 4` output back to `Refine Generated Output for Whatsapp`.  

7. **Testing and Deployment:**  
   - Deploy webhook URL in Twilio WhatsApp sandbox or production number.  
   - Verify API credentials and permissions for Perplexity, Anthropic, and Twilio.  
   - Test with sample WhatsApp queries and observe message flow and formatting.  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                             | Context or Link                                                                                              |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------|
| The workflow demonstrates a seamless 4-step process: Catch (Webhook), Research (Perplexity Sonar), Polish (Claude), Reply (Twilio WhatsApp).                                                                                            | Summary and architecture explanation in Sticky Note4.                                                       |
| The Perplexity AI prompt enforces a strict Markdown format for easy parsing and later formatting.                                                                                                                                       | See "Perform Deep Research" node prompt details.                                                             |
| Claude model is used as a stylistic editor rather than a content generator to ensure factual integrity and message brevity tailored to WhatsApp constraints (1600 chars).                                                             | See "Refine Generated Output for Whatsapp" node prompt and settings.                                         |
| Twilio messaging node swaps sender and recipient numbers automatically to ensure replies are sent correctly back to the user.                                                                                                         | See "Send Output in Whatsapp" node configuration and Sticky Note.                                            |
| Security considerations: The webhook is public, so additional validation or authentication may be needed in production to prevent misuse or unauthorized access.                                                                      | Not implemented in provided workflow; consider adding IP whitelisting or token validation.                   |
| Workflow is inactive by default (`"active": false`) and requires manual activation and webhook deployment.                                                                                                                             | Workflow meta data.                                                                                           |

---

**Disclaimer:**  
The text and workflow described here originate exclusively from an automated n8n workflow integration. The process complies fully with content policies and involves only legal and public data sources. It contains no illegal, offensive, or protected content.

---