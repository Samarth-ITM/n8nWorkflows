WhatsApp Customer Support Bot with GPT-4 Mini, Google Sheets & Rapiwa API

https://n8nworkflows.xyz/workflows/whatsapp-customer-support-bot-with-gpt-4-mini--google-sheets---rapiwa-api-10716


# WhatsApp Customer Support Bot with GPT-4 Mini, Google Sheets & Rapiwa API

### 1. Workflow Overview

This workflow implements a **WhatsApp Customer Support Bot** powered by an AI assistant named **Rapiwa**, which uses GPT-4 Mini within n8n. It integrates tightly with **Google Sheets** for dynamic product, service, and support data, and leverages multiple HTTP API tools to access external documentation portals. Incoming WhatsApp messages trigger the bot, which processes, understands, and replies with WhatsApp-compatible messages while logging interactions automatically.

**Target Use Cases:**
- Automated customer support on WhatsApp for businesses with multiple products and services.
- Quick retrieval of product/service info and company details.
- Logging customer issues and solutions for analytics and support tracking.
- Providing references to external documentation portals for detailed queries.

**Logical Blocks:**

- **1.1 Trigger & Input Validation:**  
  Starts with receiving WhatsApp messages from Rapiwa API and checks message content.

- **1.2 AI Processing & Context Management:**  
  Uses GPT-4 Mini AI agent with session memory, interpreting customer queries and generating responses.

- **1.3 Research & Data Retrieval:**  
  Accesses Google Sheets (products, services, company info, support logs) and Google Docs, plus multiple HTTP API tools linked to product documentation.

- **1.4 Support Logging:**  
  Logs customer issues, categories, and solutions into Google Sheets for record-keeping.

- **1.5 Response Dispatch:**  
  Sends the AI-generated response back to the user via Rapiwa WhatsApp API.

- **1.6 Documentation Reference Tools:**  
  Various HTTP request nodes connecting to external product documentation portals for enriched answers.

---

### 2. Block-by-Block Analysis

#### 1.1 Trigger & Input Validation

**Overview:**  
This block activates the workflow upon receiving a message from WhatsApp via the Rapiwa API and verifies that the incoming message contains text before processing further.

**Nodes Involved:**  
- Rapiwa Trigger  
- If (check text)

**Node Details:**

- **Rapiwa Trigger**  
  - *Type:* Rapiwa webhook trigger node  
  - *Configuration:* Listens to incoming WhatsApp messages via Rapiwa API credentials  
  - *Input:* Incoming Rapiwa WhatsApp event  
  - *Output:* JSON containing message and contact details  
  - *Potential Failures:* API auth errors, webhook setup issues, network timeouts  
  - *Sticky Note:* Installation guide link for Rapiwa node (https://www.npmjs.com/package/n8n-nodes-rapiwa)

- **If (check text)**  
  - *Type:* Conditional node  
  - *Configuration:* Checks if the message field (`$json.message`) is non-empty (strict string validation)  
  - *Input:* Output from Rapiwa Trigger node  
  - *Output:* Passes non-empty text messages to next block; filters others out  
  - *Edge Cases:* Non-text messages (images, audio, etc.) get blocked or require separate handling  
  - *Sticky Note:* Clarifies this node checks for text messages only

---

#### 1.2 AI Processing & Context Management

**Overview:**  
This core logic block runs the AI agent to interpret the user’s message, maintain session context, and generate an empathetic, WhatsApp-ready reply.

**Nodes Involved:**  
- AI Agent - Customer Support Agent  
- Memory  
- Think  
- OpenAI

**Node Details:**

- **AI Agent - Customer Support Agent**  
  - *Type:* LangChain AI agent node  
  - *Configuration:*  
    - Uses GPT-4 Mini model to process incoming message (`{{$json.message}}`).  
    - System message includes detailed assistant persona "Rapiwa," with instructions to access Google Sheets, Docs, and provide WhatsApp-ready plain text responses only.  
  - *Input:* Message text from "If (check text)" node and session memory from "Memory" node, plus Research outputs  
  - *Output:* Generated response text for WhatsApp  
  - *Edge Cases:* AI model timeouts, token limits, malformed inputs  
  - *Sticky Note:* Describes AI agent role as core understanding and response generator

- **Memory**  
  - *Type:* LangChain memory buffer window node  
  - *Configuration:* Uses message ID as session key, stores last 50 messages for context  
  - *Input:* Incoming message ID from trigger  
  - *Output:* Contextual memory passed to AI agent  
  - *Edge Cases:* Memory overflow, mismatched session keys

- **Think**  
  - *Type:* LangChain think tool node  
  - *Configuration:* Contains assistant persona and instructions for interpreting user requests, product info retrieval, support handling, logging, and WhatsApp formatting  
  - *Input:* Internal to AI agent for reasoning support  
  - *Output:* Thought process aiding agent’s understanding  
  - *Edge Cases:* Expression or logic failures in complex instructions

- **OpenAI**  
  - *Type:* LangChain OpenAI chat node  
  - *Configuration:* GPT-4 Mini model used as language model backend for AI agent  
  - *Input:* Messages and system prompt from AI agent  
  - *Output:* Generated text responses  
  - *Edge Cases:* API rate limits, authentication failures

---

#### 1.3 Research & Data Retrieval

**Overview:**  
This block provides the AI agent with data access tools to query Google Sheets, Google Docs, and multiple HTTP documentation portals to enrich responses.

**Nodes Involved:**  
- Research  
- Read Product  
- Read Service  
- Read Company Information  
- Docs  
- Support Desk  
- salebot  
- delix  
- socialvibe  
- faculty  
- yoori  
- meetair  
- oxoo  
- flixoo  
- OpenAI1

**Node Details:**

- **Research**  
  - *Type:* LangChain agent tool node  
  - *Configuration:* AI-driven tool that queries multiple data sources transparently for the AI agent  
  - *Input:* User chat input text  
  - *Output:* Retrieved knowledge snippets for AI agent use  
  - *Sticky Note:* Explains purpose as AI support tool hiding data sources from users

- **Read Product, Read Service, Read Company Information**  
  - *Type:* Google Sheets nodes  
  - *Configuration:* Reads product, service, and company info sheets from a shared Google Sheets document using OAuth2 credentials  
  - *Input:* Accessed by AI agent via Research node  
  - *Output:* Data rows for AI queries  
  - *Edge Cases:* Google Sheets API quota limits, missing or malformed data

- **Docs**  
  - *Type:* Google Docs node  
  - *Configuration:* Retrieves company documentation from a specified Google Docs URL  
  - *Input:* Requested by AI agent for company info  
  - *Output:* Plain text content  
  - *Edge Cases:* OAuth issues, document permissions

- **Support Desk & HTTP Request Nodes (salebot, delix, socialvibe, faculty, yoori, meetair, oxoo, flixoo)**  
  - *Type:* HTTP Request nodes  
  - *Configuration:* Each points to a specific product or service documentation URL, providing knowledge base links or API info  
  - *Input:* Accessed by Research node as auxiliary info sources  
  - *Output:* Documentation content or links  
  - *Sticky Note:* Collective sticky note above these nodes listing all product online documentation links  
  - *Edge Cases:* Network timeouts, endpoint changes, 404 errors

- **OpenAI1**  
  - *Type:* LangChain OpenAI chat node (GPT-4 Mini)  
  - *Purpose:* Backend for Research node AI processing

---

#### 1.4 Support Logging

**Overview:**  
This block logs customer issues, categories, and suggested solutions into Google Sheets, enabling tracking and analysis of support interactions.

**Nodes Involved:**  
- Log Customer Issues

**Node Details:**

- **Log Customer Issues**  
  - *Type:* Google Sheets append operation node  
  - *Configuration:* Writes new rows into a “support” sheet within the main Google Sheets doc  
  - *Columns:* SL (auto-increment), Issue, Category, Solution — values are populated from AI agent’s output using expressions `$fromAI()`  
  - *Input:* AI agent outputs issue description, category, and solution  
  - *Output:* Confirmation of log entry  
  - *Edge Cases:* Google Sheets API write errors, data mapping issues  
  - *Sticky Note:* Notes this sheet contains product details (nearby sticky note)

---

#### 1.5 Response Dispatch

**Overview:**  
This block sends the AI-generated WhatsApp-ready message back to the user via Rapiwa API.

**Nodes Involved:**  
- Replay

**Node Details:**

- **Replay**  
  - *Type:* Rapiwa API node for sending messages  
  - *Configuration:*  
    - Uses contact ID from the trigger node (`$('Rapiwa Trigger').item.json.contact_id`)  
    - Message text from AI agent output (`$json.output`)  
    - Authenticated using same Rapiwa OAuth credentials  
  - *Input:* AI agent’s generated message  
  - *Output:* Confirmation of sent message or error  
  - *Sticky Note:* Installation guide link for Rapiwa node  
  - *Edge Cases:* API rate limits, message formatting errors, connectivity issues

---

#### 1.6 Documentation Reference Tools (Sticky Notes)

**Overview:**  
Sticky notes provide inline documentation, installation instructions, and workflow explanation for maintainers and users.

**Nodes Involved:**  
- Multiple Sticky Notes scattered around the workflow

**Node Details:**

- Sticky Note near Rapiwa Trigger: Installation hint for Rapiwa nodes with link  
- Sticky Note near If (check text): Explains text message validation  
- Sticky Note near AI Agent: Describes assistant persona and core AI function  
- Sticky Note near Research node: Explains AI support tool role  
- Sticky Note near Replay: Mentions WhatsApp reply via Rapiwa API with installation link  
- Large Sticky Note (near left side): Full workflow overview, key features, integration details, and useful links  
- Sticky Note above HTTP request nodes: Lists all product online documentation links  
- Sticky Note near Log Customer Issues: Notes about product details stored on sheet  
- Sticky Note near “Example product link” over Read Company Info node

---

### 3. Summary Table

| Node Name                  | Node Type                          | Functional Role                         | Input Node(s)             | Output Node(s)               | Sticky Note                                                                                   |
|----------------------------|----------------------------------|---------------------------------------|---------------------------|-----------------------------|-----------------------------------------------------------------------------------------------|
| Rapiwa Trigger             | Rapiwa Trigger                   | Entry point: receives WhatsApp messages | -                         | If (check text)             | Installation guide: [how to install rapiwa](https://www.npmjs.com/package/n8n-nodes-rapiwa)   |
| If (check text)            | If                              | Validates message contains text       | Rapiwa Trigger            | AI Agent - Customer Support Agent | Checks if message text is non-empty                                                           |
| AI Agent - Customer Support Agent | LangChain agent                 | Main AI logic: interprets message, generates response | If (check text), Memory, Research, Think, OpenAI | Replay                       | Core AI assistant logic and WhatsApp-ready response generation                                |
| Memory                     | LangChain memoryBufferWindow      | Stores session context for chat       | Rapiwa Trigger (message_id) | AI Agent - Customer Support Agent | Session memory management                                                                     |
| Think                      | LangChain toolThink               | Internal AI assistant reasoning helper | -                         | AI Agent - Customer Support Agent | Contains assistant persona and instructions                                                  |
| OpenAI                     | LangChain lmChatOpenAi            | GPT-4 Mini language model backend     | AI Agent - Customer Support Agent | AI Agent - Customer Support Agent | Uses GPT-4 Mini                                                                              |
| Research                   | LangChain agentTool               | Data retrieval tool for AI agent      | Docs, Read Product, Read Service, Read Company Information, Support Desk, salebot, delix, socialvibe, faculty, yoori, meetair, oxoo, flixoo, OpenAI1 | AI Agent - Customer Support Agent | Aggregates data from Sheets, Docs, HTTP docs, and AI backend                                |
| Read Product               | Google Sheets Tool                | Fetch product data                    | -                         | Research                    | Reads product info from Google Sheets                                                        |
| Read Service               | Google Sheets Tool                | Fetch service data                    | -                         | Research                    | Reads service info from Google Sheets                                                        |
| Read Company Information   | Google Sheets Tool                | Fetch company info                    | -                         | Research                    | Reads company info from Google Sheets                                                        |
| Docs                       | Google Docs Tool                 | Fetch company documentation           | -                         | Research                    | Retrieves Google Docs content                                                                |
| Support Desk               | HTTP Request Tool                | Knowledge base API access              | -                         | Research                    | Links to knowledge base docs                                                                 |
| salebot                    | HTTP Request Tool                | Product documentation link             | -                         | Research                    | [https://docs.salebot.app/](https://docs.salebot.app/)                                       |
| delix                      | HTTP Request Tool                | Product documentation link             | -                         | Research                    | [https://docs.delix.cloud/](https://docs.delix.cloud/)                                       |
| socialvibe                 | HTTP Request Tool                | Product documentation link             | -                         | Research                    | [https://socialvibe.spagreen.net/docs/](https://socialvibe.spagreen.net/docs/)               |
| faculty                    | HTTP Request Tool                | Product documentation link             | -                         | Research                    | [https://faculty.spagreen.net/docs/](https://faculty.spagreen.net/docs/)                     |
| yoori                      | HTTP Request Tool                | Product documentation link             | -                         | Research                    | [https://docs.spagreen.net/docs/yoori/get-started/introduction](https://docs.spagreen.net/docs/yoori/get-started/introduction) |
| meetair                    | HTTP Request Tool                | Product documentation link             | -                         | Research                    | [https://meetair.spagreen.net/docs/](https://meetair.spagreen.net/docs/)                     |
| oxoo                       | HTTP Request Tool                | Product documentation link             | -                         | Research                    | [https://oxoo.spagreen.net/documentation/android/](https://oxoo.spagreen.net/documentation/android/) |
| flixoo                     | HTTP Request Tool                | Product documentation link             | -                         | Research                    | [https://docs.flixoo.app/](https://docs.flixoo.app/)                                         |
| OpenAI1                    | LangChain lmChatOpenAi            | GPT-4 Mini for Research node          | Research                  | Research                    | GPT-4 Mini AI backend for Research node                                                     |
| Log Customer Issues        | Google Sheets Tool                | Logs customer issues and solutions    | AI Agent - Customer Support Agent | Research                    | Stores issues, categories, and solutions in Google Sheets                                    |
| Replay                     | Rapiwa API node                  | Sends WhatsApp message reply          | AI Agent - Customer Support Agent | -                           | Installation guide: [how to install rapiwa](https://www.npmjs.com/package/n8n-nodes-rapiwa)   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Rapiwa Trigger node:**  
   - Type: `Rapiwa Trigger`  
   - Credentials: Configure with Rapiwa API OAuth2 credentials  
   - Purpose: Receive WhatsApp messages  
   - Save and set webhook URL in Rapiwa dashboard

2. **Add If node (check text):**  
   - Type: `If`  
   - Condition: Check if `{{$json.message}}` is not empty string (strict string validation)  
   - Connect input from `Rapiwa Trigger` output  
   - Pass text messages only

3. **Add LangChain Memory node:**  
   - Type: `LangChain memoryBufferWindow`  
   - Session Key: `{{$json.message_id}}` (custom key)  
   - Context Window Length: 50  
   - No input connections (used internally)

4. **Add LangChain toolThink node (Think):**  
   - Type: `LangChain toolThink`  
   - Paste assistant persona and instructions as provided (describing Rapiwa’s role, data access rules, response style)  

5. **Add LangChain OpenAI node (OpenAI):**  
   - Type: `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
   - Model: GPT-4 Mini (`gpt-4.1-mini`)

6. **Add LangChain AI Agent node (AI Agent - Customer Support Agent):**  
   - Type: `@n8n/n8n-nodes-langchain.agent`  
   - Text: `={{ $json.message }}`  
   - System Message: Paste same assistant persona/instructions as in Think node  
   - Prompt Type: define  
   - Connect inputs:  
     - Message from `If (check text)` node  
     - Memory from `Memory` node (ai_memory input)  
     - Think node (ai_tool input)  
     - OpenAI node (ai_languageModel input)  
     - Research node (ai_tool input, created next)

7. **Add Research node:**  
   - Type: `@n8n/n8n-nodes-langchain.agentTool`  
   - Text: `={{ $('Rapiwa Trigger').item.json.message }}` or equivalent user input  
   - Tool Description: Paste assistant persona with data source references  
   - Connect inputs:  
     - Google Sheets nodes (products, services, company info)  
     - Google Docs node  
     - Multiple HTTP Request nodes (support desk and product docs)  
     - OpenAI1 node (GPT-4 Mini model for Research)

8. **Add Google Sheets nodes:**  
   - Read Product: Read from products sheet (gid=0)  
   - Read Service: Read from service sheet (specified gid)  
   - Read Company Information: Read from company info sheet (specified gid)  
   - Log Customer Issues: Append mode to support log sheet with columns SL, Issue, Category, Solution using expressions from AI agent output

9. **Add Google Docs node:**  
   - Configure with OAuth2 credentials  
   - Document URL: Add your company info Google Docs link

10. **Add HTTP Request nodes for documentation portals:**  
    - For each URL (salebot, delix, socialvibe, faculty, yoori, meetair, oxoo, flixoo)  
    - Set URL accordingly and tool description with link

11. **Add OpenAI node (OpenAI1):**  
    - Same GPT-4 Mini model for research AI backend

12. **Connect all data nodes (Google Sheets, Google Docs, HTTP requests, OpenAI1) as inputs to Research node**

13. **Connect Research node output to AI Agent node (ai_tool input)**

14. **Connect AI Agent node main output to Replay node**

15. **Add Replay node:**  
    - Type: Rapiwa API node for sending messages  
    - Number: `={{ $('Rapiwa Trigger').item.json.contact_id }}`  
    - Message: `={{ $json.output }}` from AI Agent  
    - Credentials: Same Rapiwa OAuth2 credentials

16. **Connect Replay node output to end**

17. **Add Sticky Notes as needed for documentation and maintenance**

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                         | Context or Link                                                                                     |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| Workflow designed for WhatsApp AI customer support using Rapiwa API, Google Sheets, Google Docs, and GPT-4 Mini AI integration within n8n.                                                                                                                                                                                                                                                           | Overall workflow purpose                                                                            |
| Installation guide for Rapiwa nodes: [https://www.npmjs.com/package/n8n-nodes-rapiwa](https://www.npmjs.com/package/n8n-nodes-rapiwa)                                                                                                                                                                                                                                                                   | Sticky notes near Rapiwa Trigger and Replay nodes                                                  |
| Product documentation portals: SaleBot, Delix, SocialVibe, Faculty, Yoori, MeetAir, Oxoo, Flixoo — URLs embedded in HTTP Request nodes for enriched user responses.                                                                                                                                                                                                                                     | Sticky note near HTTP Request nodes                                                                |
| Google Sheets document holds product, service, company info, and support logs; Google Docs provides company documentation. OAuth2 credentials required for both.                                                                                                                                                                                                                                      | Credentials setup for Google Sheets and Docs                                                       |
| AI assistant "Rapiwa" persona emphasizes empathy, professionalism, conversational tone, WhatsApp message formatting (plain text, URLs, mobile-friendly), and never revealing internal data sources to end users.                                                                                                                                                                                      | System message in AI Agent and Think nodes                                                         |
| Useful community and support links: Discord SpaGreen Community, Facebook SpaGreen Support Group, official websites for Rapiwa and SpaGreen.                                                                                                                                                                                                                                                          | Large sticky note on left side of workflow                                                        |

---

**Disclaimer:** The provided text is derived exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly accessible.