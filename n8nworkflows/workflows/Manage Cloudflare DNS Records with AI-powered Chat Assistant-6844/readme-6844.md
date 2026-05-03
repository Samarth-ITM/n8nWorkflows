Manage Cloudflare DNS Records with AI-powered Chat Assistant

https://n8nworkflows.xyz/workflows/manage-cloudflare-dns-records-with-ai-powered-chat-assistant-6844


# Manage Cloudflare DNS Records with AI-powered Chat Assistant

### 1. Workflow Overview

This workflow enables AI-powered management of Cloudflare DNS records via a chat interface. It targets users who want to interact with their Cloudflare DNS zones through natural language queries, leveraging a large language model (LLM) to interpret commands and perform DNS operations such as listing domains, retrieving DNS records, and updating DNS entries.

The workflow is logically divided into the following blocks:

- **1.1 Chat Message Reception**: Listens for incoming chat messages from users.
- **1.2 AI Processing & Chat Agent**: Uses an LLM (OpenAI GPT-4o-mini) combined with a custom chat agent to parse user intent and decide on required Cloudflare operations.
- **1.3 Cloudflare API Operations**: Performs DNS management actions (list domains, get DNS records, set DNS records) by calling Cloudflare APIs.
- **1.4 Data Parsing and Routing**: Processes responses and routes data based on action types.
- **1.5 Auxiliary Nodes**: Includes notes, sticky notes with documentation, and a no-operation end node.

---

### 2. Block-by-Block Analysis

#### 2.1 Chat Message Reception

- **Overview:**  
  This block waits for incoming chat messages to trigger the workflow.

- **Nodes Involved:**  
  - When chat message received

- **Node Details:**

  - **When chat message received**  
    - *Type and Role:* LangChain Chat Trigger node; entry point that listens for incoming chat messages.  
    - *Configuration:* Default settings with webhook enabled (webhookId provided).  
    - *Expressions / Variables:* None; triggers on any incoming chat message.  
    - *Input / Output:* No input; outputs chat message data downstream.  
    - *Version:* 1.1  
    - *Failure Modes:* Network issues, webhook misconfiguration, or invalid payloads could drop triggers.

---

#### 2.2 AI Processing & Chat Agent

- **Overview:**  
  Processes the chat input using an OpenAI chat model and a specialized LangChain chat agent configured to manage Cloudflare DNS operations. Integrates a PostgreSQL memory node to maintain chat history context.

- **Nodes Involved:**  
  - OpenAI Chat Model  
  - Chat Agent  
  - Postgres Chat Memory  
  - CloudFlare tool  
  - End

- **Node Details:**

  - **OpenAI Chat Model**  
    - *Type and Role:* LLM chat node using OpenAI GPT-4o-mini to process natural language input.  
    - *Configuration:* Uses GPT-4o-mini model; OpenAI API credentials configured.  
    - *Expressions / Variables:* None directly; receives raw user chat input.  
    - *Input / Output:* Input from chat trigger; output to Chat Agent node.  
    - *Version:* 1.2  
    - *Failure Modes:* API limits, invalid credentials, timeout, or model-specific errors.

  - **Chat Agent**  
    - *Type and Role:* LangChain agent node; orchestrates the logic for DNS management commands by interpreting user intent.  
    - *Configuration:*  
      System message defines the agent as a DNS manager for Cloudflare with access to a custom tool ("cf_tool") for fetching and setting DNS records. Defines clear instructions for handling three actions: 'get_dns', 'set_dns', and 'domain'.  
    - *Expressions / Variables:* System message contains dynamic instruction text and usage rules for calling the "cf_tool".  
    - *Input / Output:* Input from OpenAI Chat Model and memory node; output to End node.  
    - *Version:* 1.9  
    - *Failure Modes:* Misinterpretation of user input, failure to call sub-workflow tool, or tool response errors.

  - **Postgres Chat Memory**  
    - *Type and Role:* Stores chat histories in a PostgreSQL database for context retention across conversations.  
    - *Configuration:* Uses table "langchain_chat_histories"; context window length set to 6 messages for context.  
    - *Input / Output:* Inputs chat data; outputs enriched chat context to Chat Agent.  
    - *Version:* 1.3  
    - *Failure Modes:* DB connectivity, query failures, or data corruption.

  - **CloudFlare tool**  
    - *Type and Role:* LangChain tool workflow node that delegates Cloudflare operations to a sub-workflow (the current workflow itself).  
    - *Configuration:* Points to current workflow ID for delegated execution; expects JSON input with action parameters.  
    - *Input / Output:* Accepts JSON input from Chat Agent; outputs results back to Chat Agent.  
    - *Version:* 2.2  
    - *Failure Modes:* Recursive workflow invocation limits, invalid input parameters, or runtime errors in downstream nodes.

  - **End**  
    - *Type and Role:* No-operation node; marks the end of the AI processing chain.  
    - *Configuration:* None.  
    - *Input / Output:* Input from Chat Agent; no outputs.  
    - *Version:* 1  
    - *Failure Modes:* None.

---

#### 2.3 Cloudflare API Operations

- **Overview:**  
  This block executes actual HTTP requests to Cloudflare's API to retrieve or modify DNS data based on the action specified by the AI agent.

- **Nodes Involved:**  
  - Json  
  - Switch  
  - Get TLDs  
  - Host details  
  - Getter  
  - Setter  
  - SubCall

- **Node Details:**

  - **SubCall**  
    - *Type and Role:* Execute Workflow Trigger; entry point for tool workflow calls.  
    - *Configuration:* Default; invoked by CloudFlare tool node.  
    - *Input / Output:* Receives JSON commands; outputs to Json node.  
    - *Version:* 1  
    - *Failure Modes:* Workflow execution failures, invalid input.

  - **Json**  
    - *Type and Role:* Code node; parses incoming JSON command strings to objects for routing.  
    - *Configuration:* JavaScript code attempts to parse `query` from input JSON, extracts `action` field if present.  
    - *Expressions / Variables:* Uses `$input.first().json.query` to access input; returns parsed object with `action` key.  
    - *Input / Output:* Input from SubCall; output to Switch.  
    - *Version:* 2  
    - *Failure Modes:* JSON parse errors, missing expected fields.

  - **Switch**  
    - *Type and Role:* Routes workflow based on `action` field in JSON (`domain`, `get_dns`, `set_dns`).  
    - *Configuration:* Three routing rules matching `domain`, `get_dns`, and `set_dns`.  
    - *Expressions / Variables:* Uses expression `={{ $json.out.action }}` for routing.  
    - *Input / Output:* Input from Json; outputs to Get TLDs, Getter, or Setter nodes respectively.  
    - *Version:* 3.2  
    - *Failure Modes:* Missing or malformed `action` values causing routing failures.

  - **Get TLDs**  
    - *Type and Role:* HTTP Request; fetches Cloudflare DNS zones (domains).  
    - *Configuration:* GET request to `https://api.cloudflare.com/client/v4/zones` with `per_page=100` query parameter.  
    - *Authentication:* Uses custom HTTP authentication credentials (Cloudflare API token/key).  
    - *Input / Output:* Input from Switch; outputs response to Host details node.  
    - *Version:* 4.1  
    - *Failure Modes:* API key issues, rate limits, network errors.

  - **Host details**  
    - *Type and Role:* SplitOut node; extracts `result` array from API response JSON.  
    - *Configuration:* Splits out the `result` field to separate items.  
    - *Input / Output:* Input from Get TLDs; outputs individual domain objects.  
    - *Version:* 1  
    - *Failure Modes:* Missing `result` field, unexpected response structure.

  - **Getter**  
    - *Type and Role:* HTTP Request; retrieves DNS records for a specified domain (`domain_id`).  
    - *Configuration:* GET request to `https://api.cloudflare.com/client/v4/zones/{{ $json.out.domain_id }}/dns_records`.  
    - *Authentication:* Cloudflare API credentials used.  
    - *Input / Output:* Input from Switch; outputs DNS records JSON.  
    - *Version:* 4.1  
    - *Failure Modes:* Invalid domain ID, auth errors, API or network errors.

  - **Setter**  
    - *Type and Role:* HTTP Request; updates a specific DNS record on Cloudflare.  
    - *Configuration:* PATCH request to `https://api.cloudflare.com/client/v4/zones/{{ $json.out.domain_id }}/dns_records/{{ $json.out.record_id }}` with body parameter `content` set to new record content.  
    - *Authentication:* Cloudflare API credentials used.  
    - *Input / Output:* Input from Switch; outputs updated record response.  
    - *Version:* 4.1  
    - *Failure Modes:* Invalid record ID, permission errors, malformed update content.

---

#### 2.4 Auxiliary Nodes

- **Overview:**  
  Contains documentation and informational sticky notes to assist users in understanding prerequisites, author info, and support resources.

- **Nodes Involved:**  
  - Sticky Note2  
  - Sticky Note5  
  - Sticky Note6  
  - Sticky Note8  
  - Sticky Note9  
  - Note1  
  - Note2

- **Node Details:**

  - **Sticky Note2**  
    - *Content:* Lists requirements: Cloudflare API key/token and OpenAI credentials.  
    - *Role:* Informational.

  - **Sticky Note5**  
    - *Content:* Author info with avatar, LinkedIn, video overview, and creator profile links.  
    - *Role:* Attribution and resources.

  - **Sticky Note6**  
    - *Content:* Help resource link to n8n community forum.  
    - *Role:* Support.

  - **Sticky Note8**  
    - *Content:* Empty sticky note with color styling.  
    - *Role:* Visual separation.

  - **Sticky Note9**  
    - *Content:* Title "CloudFlare chat ↪️️ get / set DNS records".  
    - *Role:* Section header.

  - **Note1**  
    - *Content:* Describes the chat step using any LLM; tested with Gemini and GPT 4o-mini; explains prompt design for DNS operations.  
    - *Role:* Guidance for chat model usage.

  - **Note2**  
    - *Content:* Describes main Cloudflare operations (get/set DNS records).  
    - *Role:* Operational context.

---

### 3. Summary Table

| Node Name                | Node Type                                    | Functional Role                         | Input Node(s)               | Output Node(s)                          | Sticky Note                                                    |
|--------------------------|----------------------------------------------|---------------------------------------|----------------------------|---------------------------------------|----------------------------------------------------------------|
| When chat message received | @n8n/n8n-nodes-langchain.chatTrigger        | Entry point: listens for chat messages | None                       | OpenAI Chat Model                      |                                                                |
| OpenAI Chat Model         | @n8n/n8n-nodes-langchain.lmChatOpenAi        | Processes user input with GPT-4o-mini | When chat message received  | Chat Agent                            |                                                                |
| Chat Agent               | @n8n/n8n-nodes-langchain.agent                | Interprets intent; calls Cloudflare tool | OpenAI Chat Model, Postgres Chat Memory, CloudFlare tool | End                                   |                                                                |
| Postgres Chat Memory      | @n8n/n8n-nodes-langchain.memoryPostgresChat  | Maintains chat history context        | None                       | Chat Agent                            |                                                                |
| CloudFlare tool           | @n8n/n8n-nodes-langchain.toolWorkflow         | Delegates Cloudflare API calls        | Chat Agent                 | Chat Agent                            |                                                                |
| End                      | n8n-nodes-base.noOp                           | Marks end of AI processing             | Chat Agent                 | None                                 |                                                                |
| SubCall                  | n8n-nodes-base.executeWorkflowTrigger         | Entry point for Cloudflare operations  | CloudFlare tool            | Json                                 |                                                                |
| Json                     | n8n-nodes-base.code                           | Parses JSON command and extracts action | SubCall                   | Switch                               |                                                                |
| Switch                   | n8n-nodes-base.switch                         | Routes by action type                   | Json                       | Get TLDs, Getter, Setter             |                                                                |
| Get TLDs                 | n8n-nodes-base.httpRequest                     | Retrieves Cloudflare domains           | Switch                     | Host details                        |                                                                |
| Host details             | n8n-nodes-base.splitOut                         | Extracts domain list from API response | Get TLDs                   | None (final output)                   |                                                                |
| Getter                   | n8n-nodes-base.httpRequest                     | Retrieves DNS records for a domain     | Switch                     | None (final output)                   |                                                                |
| Setter                   | n8n-nodes-base.httpRequest                     | Updates DNS record content              | Switch                     | None (final output)                   |                                                                |
| Sticky Note2              | n8n-nodes-base.stickyNote                      | Documents requirements                  | None                       | None                                 | For storing and processing of data in this flow you will need: CloudFlare API key/token and OpenAI credentials saved. |
| Sticky Note5              | n8n-nodes-base.stickyNote                      | Author information and resource links  | None                       | None                                 | Author: Kresimir Pendic; Video: https://youtu.be/bqrxn_41K1Y; LinkedIn: https://www.linkedin.com/in/mkdizajn/; Templates: https://n8n.io/creators/kres/ |
| Sticky Note6              | n8n-nodes-base.stickyNote                      | Help and community forum link           | None                       | None                                 | For help, create topic at https://community.n8n.io/c/questions/ |
| Sticky Note8              | n8n-nodes-base.stickyNote                      | Visual separation                       | None                       | None                                 |                                                                |
| Sticky Note9              | n8n-nodes-base.stickyNote                      | Section header                         | None                       | None                                 | CloudFlare chat ↪️️ get / set DNS records                     |
| Note1                     | n8n-nodes-base.stickyNote                      | Chat step explanation                   | None                       | None                                 | Use any LLM (tested Gemini, GPT 4o-mini). Main TLD DNS ops prepared with easy prompt. |
| Note2                     | n8n-nodes-base.stickyNote                      | Cloudflare operation context            | None                       | None                                 | Main operation call for getting, setting DNS records          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add the entry trigger node:**  
   - Add node: `@n8n/n8n-nodes-langchain.chatTrigger`  
   - Name it: "When chat message received"  
   - Configure webhook settings as needed (default is fine).  
   - This node will start the workflow on incoming chat messages.

3. **Add the OpenAI chat model node:**  
   - Add node: `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
   - Name it: "OpenAI Chat Model"  
   - Set model to "gpt-4o-mini".  
   - Configure OpenAI credentials (add your OpenAI API key).  
   - Connect: "When chat message received" → "OpenAI Chat Model".

4. **Add PostgreSQL chat memory node:**  
   - Add node: `@n8n/n8n-nodes-langchain.memoryPostgresChat`  
   - Name it: "Postgres Chat Memory"  
   - Set table name to "langchain_chat_histories".  
   - Set context window length to 6.  
   - Configure PostgreSQL credentials.  
   - Connect "OpenAI Chat Model" output into this node. (Alternatively, connect as ai_memory input to Chat Agent.)

5. **Add the Chat Agent node:**  
   - Add node: `@n8n/n8n-nodes-langchain.agent`  
   - Name it: "Chat Agent"  
   - Configure system message with instructions:  
     ```
     You are dns manager for CloudFlare

     You have cf_tool that is your source for getting and setting DNS records.
     Only return answers based from cf_tool.

     When calling cf_tool, data args to pass for actions are:

     If you need to get dns record, call cf_tool and pass:
     - action: 'get_dns'
     - domain_id: domain id

     if you need to set dns record, call cf_tool and pass:
     - action: 'set_dns'
     - domain_id: domain id
     - record_id: id of a record
     - record_type: type of DNS record
     - new_record_content: new value

     If you need to get domains available in CF, call cf_tool and pass:
     - action 'domain'

     Before calling any user query action, first execute get all domains action.
     ```
   - Connect outputs:  
     - `ai_languageModel` input from "OpenAI Chat Model"  
     - `ai_memory` input from "Postgres Chat Memory"  
     - `ai_tool` input from CloudFlare tool (to be created next)  
   - Connect output to "End" node (to be created).

6. **Add CloudFlare tool node:**  
   - Add node: `@n8n/n8n-nodes-langchain.toolWorkflow`  
   - Name it: "CloudFlare tool"  
   - Configure to call this workflow itself (set workflow ID of the current workflow).  
   - Set input mapping mode to "defineBelow" with an empty schema (accepts any JSON).  
   - Connect output to Chat Agent `ai_tool` input.

7. **Add End node:**  
   - Add node: `n8n-nodes-base.noOp`  
   - Name it: "End"  
   - Connect input from Chat Agent main output.

8. **Add Sub-workflow entry node:**  
   - Add node: `n8n-nodes-base.executeWorkflowTrigger`  
   - Name it: "SubCall"  
   - This node will be called by the CloudFlare tool node.  
   - Connect output to next node "Json".

9. **Add Json parsing node:**  
   - Add node: `n8n-nodes-base.code`  
   - Name it: "Json"  
   - JavaScript code:  
     ```javascript
     let inpt = $input.first().json.query;
     let out = {};
     try{
       out = JSON.parse(inpt)
     } catch{}
     if( out.action ){
       return {out};
     } else{
       return {};
     }
     ```
   - Connect from "SubCall" to "Json".

10. **Add Switch node for action routing:**  
    - Add node: `n8n-nodes-base.switch`  
    - Name it: "Switch"  
    - Create three rules based on `{{$json.out.action}}`:  
      - Equals "domain" → output 1  
      - Equals "get_dns" → output 2  
      - Equals "set_dns" → output 3  
    - Connect "Json" output to "Switch".

11. **Add Get TLDs node:**  
    - Add node: `n8n-nodes-base.httpRequest`  
    - Name it: "Get TLDs"  
    - Method: GET  
    - URL: `https://api.cloudflare.com/client/v4/zones`  
    - Query parameter: `per_page=100`  
    - Authentication: HTTP Custom Auth with Cloudflare API token or key.  
    - Connect "Switch" output 1 (domain) to "Get TLDs".

12. **Add Host details node:**  
    - Add node: `n8n-nodes-base.splitOut`  
    - Name it: "Host details"  
    - Field to split out: `result`  
    - Connect "Get TLDs" output to "Host details".

13. **Add Getter node:**  
    - Add node: `n8n-nodes-base.httpRequest`  
    - Name it: "Getter"  
    - Method: GET  
    - URL: `https://api.cloudflare.com/client/v4/zones/{{ $json.out.domain_id }}/dns_records`  
    - Authentication: HTTP Custom Auth with Cloudflare API credentials.  
    - Connect "Switch" output 2 (get_dns) to "Getter".

14. **Add Setter node:**  
    - Add node: `n8n-nodes-base.httpRequest`  
    - Name it: "Setter"  
    - Method: PATCH  
    - URL: `https://api.cloudflare.com/client/v4/zones/{{ $json.out.domain_id }}/dns_records/{{ $json.out.record_id }}`  
    - Body parameter: `content` with value `{{$json.out.new_record_content}}`  
    - Authentication: HTTP Custom Auth with Cloudflare API credentials.  
    - Connect "Switch" output 3 (set_dns) to "Setter".

15. **Add Sticky Notes and Documentation nodes:**  
    - Add the sticky notes with content as per Sticky Note2, Sticky Note5, Sticky Note6, Sticky Note8, Sticky Note9, Note1, and Note2 for user guidance and info.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                              | Context or Link                                                                                      |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| For storing and processing of data in this flow you will need: CloudFlare.com API key/token (https://dash.cloudflare.com/?to=/:account/api-tokens) and OpenAI credentials saved. | Sticky Note2                                                                                        |
| Author: Kresimir Pendic – Senior professional specializing in automation, AI, and data analysis. Video overview: https://youtu.be/bqrxn_41K1Y. LinkedIn: https://www.linkedin.com/in/mkdizajn/. Templates: https://n8n.io/creators/kres/ | Sticky Note5                                                                                        |
| For help, create a topic on the community forums here: https://community.n8n.io/c/questions/                                                               | Sticky Note6                                                                                        |
| Chat step uses any LLM; tested with Gemini and GPT 4o-mini. Prepared main get|set TLD DNS records operations with easy prompt extendable to other CF operations.                | Note1                                                                                              |
| Main Cloudflare operation call for getting and setting DNS records.                                                                                        | Note2                                                                                              |

---

**Disclaimer:**  
The provided text is exclusively derived from an automated n8n workflow. It complies strictly with content policies and contains no illegal, offensive, or protected elements. All data processed are legal and public.