Teleport Workflows Between n8n Instances with Magic Inbox P2P System

https://n8nworkflows.xyz/workflows/teleport-workflows-between-n8n-instances-with-magic-inbox-p2p-system-6960


# Teleport Workflows Between n8n Instances with Magic Inbox P2P System

---

# Teleport Workflows Between n8n Instances with Magic Inbox P2P System

---

## 1. Workflow Overview

This workflow enables **peer-to-peer (P2P) teleportation of n8n workflows between different n8n instances** using a Magic Inbox system. It facilitates server-to-server communication via Telegram channels and Magic Inbox nodes, allowing users, creators, and teams to send, receive, and manage workflows seamlessly without intermediaries.

### Target Use Cases
- Sharing complex n8n workflows instantly between different n8n servers.
- Enabling creators to reward communities with premium workflows.
- Allowing teams to exchange workflows via chat or Telegram.
- Managing P2P workflow transfer with automated analysis, consolidation, and validation.

### Logical Blocks

- **1.1 Input Reception**: Captures incoming Telegram messages and Magic Inbox messages as triggers.
- **1.2 Dispatcher & Analysis**: Uses AI to analyze user requests and distribute tasks to specialized submodules.
- **1.3 Specialized AI Processing**: Seven specialized AI "golden_dev" agents operate on task fragments: coordination, triggering, logic, integration, data, communication, and robustness.
- **1.4 Fragment Merging & Consolidation**: Merges outputs from all specialized agents and consolidates them into a coherent single workflow.
- **1.5 Dynamic Connection Creation**: Refactors node IDs, creates logical connections, and validates the final workflow.
- **1.6 Workflow Sending**: Sends the finalized workflow back through Magic Inbox or Telegram.

---

## 2. Block-by-Block Analysis

### 1.1 Input Reception

**Overview:**  
This block captures messages from Telegram channels and Magic Inbox to start the workflow teleportation process.

**Nodes Involved:**  
- Message server n8n trigger (Telegram Trigger)  
- Receiving message server n8n trigger (Magic Inbox)  
- Receiving message server n8n (Telegram node)  
- When clicking (Manual Trigger)  

#### Node Details:

- **Message server n8n trigger**  
  - Type: Telegram Trigger  
  - Role: Listens for incoming Telegram messages in a specific Telegram group or channel.  
  - Configuration: Triggered on "message" updates, linked to a Telegram API credential named "My server group".  
  - Output: Sends message JSON forward.  
  - Edge Cases: Telegram API rate limits, webhook misconfiguration, permission errors.

- **Receiving message server n8n trigger**  
  - Type: Magic Inbox Trigger  
  - Role: Listens for incoming messages from Magic Inbox (P2P system).  
  - Configuration: Uses n8n API Key and instance URL for authentication.  
  - Output: Triggers when a workflow or message is received via Magic Inbox.  
  - Edge Cases: API key expiration, instance URL misconfiguration, network failures.

- **Receiving message server n8n**  
  - Type: Telegram (Send Message)  
  - Role: Sends formatted Telegram messages based on received Magic Inbox input.  
  - Configuration: Sends text constructed with message metadata (sender, message content, workflow import details, timestamp).  
  - Credential: Telegram API for the target chat ID.  
  - Edge Cases: Invalid chat ID, Telegram send limits.

- **When clicking**  
  - Type: Manual Trigger  
  - Role: Allows manual initiation for testing or independent workflows.  
  - No parameters configured.

---

### 1.2 Dispatcher & Analysis

**Overview:**  
This block uses a Perplexity AI node as a team leader dispatcher to analyze incoming user requests and distribute technical tasks precisely to specialized AI agents.

**Nodes Involved:**  
- timestamp_generator1 (DateTime)  
- team_leader_dispatcher1 (Perplexity AI)  
- information_extractor1 (Langchain Information Extractor, disabled)  
- OpenAI Chat Model5 (OpenAI Chat)  

#### Node Details:

- **timestamp_generator1**  
  - Type: DateTime  
  - Role: Generates current timestamp (Europe/Paris timezone) for tracking and workflow naming.  
  - Output: Adds timestamp field to JSON.  
  - Edge Cases: Incorrect timezone configuration.

- **team_leader_dispatcher1**  
  - Type: Perplexity AI  
  - Role: Analyzes the user request with a detailed system prompt to assign specific n8n node types and instructions to 7 specialized developers (golden_dev, golden_dev1...golden_dev6).  
  - Configuration: Uses "sonar-reasoning-pro" model with a complex system message instructing task distribution and node type validation.  
  - Output: JSON with node assignments and workflow implementation specifications.  
  - Edge Cases: API rate limits, malformed user input, AI model failures.

- **information_extractor1** (Disabled)  
  - Type: Langchain Information Extractor  
  - Role: Intended to parse natural language output from dispatcher into structured JSON. Currently disabled.  
  - Edge Cases: Parsing errors, schema mismatches.

- **OpenAI Chat Model5**  
  - Type: Langchain OpenAI Chat Model  
  - Role: Supports information_extractor1 with OpenAI GPT-4.1-mini model.  
  - Credentials: OpenAI API key required.  
  - Edge Cases: API key issues, request timeouts.

---

### 1.3 Specialized AI Processing

**Overview:**  
Seven specialized AI agents (named golden_dev, golden_dev1, ..., golden_dev6) each handle a specific aspect of workflow generation based on the dispatcher’s instructions, using OpenRouter Claude-sonnet-4 model.

**Nodes Involved:**  
- golden_dev (agent)  
- golden_dev1 (agent)  
- golden_dev2 (agent)  
- golden_dev3 (agent)  
- golden_dev4 (agent)  
- golden_dev5 (agent)  
- golden_dev6 (agent)  
- Think Visionnaire1 to Think Visionnaire7 (toolThink nodes)  
- OpenRouter Chat Model to OpenRouter Chat Model6 (AI language models)  

#### Node Details:

- **golden_dev (Coordination Agent)**  
  - Type: Langchain agent  
  - Role: Handles coordination tasks (merge, switch, if, wait nodes).  
  - Input: Receives prompts from team_leader_dispatcher1 via Think Visionnaire1.  
  - Output: JSON fragment with generated nodes for coordination role.  
  - Model: Anthropic Claude-sonnet-4 via OpenRouter API.  
  - Edge Cases: Model response errors, API failures.

- **golden_dev1 (Triggering Agent)**  
  - Role: Handles trigger nodes (webhook, schedule, manual).  
  - Connected to Think Visionnaire2 and OpenRouter Chat Model1.  
  - Edge Cases: Incorrect trigger configuration leading to no event firing.

- **golden_dev2 (Logic Agent)**  
  - Role: Handles logic nodes (function, code, date-time).  
  - Connected to Think Visionnaire3 and OpenRouter Chat Model2.

- **golden_dev3 (Integration Agent)**  
  - Role: Handles integration nodes (HTTP request, OAuth).  
  - Connected to Think Visionnaire4 and OpenRouter Chat Model3.

- **golden_dev4 (Data Agent)**  
  - Role: Handles data transformation nodes (set, respond to webhook).  
  - Connected to Think Visionnaire7 and OpenRouter Chat Model4.

- **golden_dev5 (Communication Agent)**  
  - Role: Handles communication nodes (email, slack, discord).  
  - Connected to Think Visionnaire5 and OpenRouter Chat Model5.

- **golden_dev6 (Robustness Agent)**  
  - Role: Handles error handling, monitoring nodes (function for error handling, HTTP request for monitoring).  
  - Connected to Think Visionnaire6 and OpenRouter Chat Model6.

- **Think Visionnaire1 to 7**  
  - Type: toolThink (Langchain utility node)  
  - Role: Intermediate thinking tools feeding prompts to agents.  
  - Edge Cases: Expression failures or blank inputs.

---

### 1.4 Fragment Merging & Consolidation

**Overview:**  
This block merges the outputs from all 7 specialized agents into a unified workflow fragment and consolidates them, preparing for final connection setup.

**Nodes Involved:**  
- merge_all_fragments2 (Merge node)  
- consolidator (Code node)  

#### Node Details:

- **merge_all_fragments2**  
  - Type: Merge (combine mode by position)  
  - Role: Combines all 7 specialized outputs into a single JSON array, handling name clashes by adding suffixes.  
  - Inputs: Receives outputs from golden_dev, golden_dev1 to golden_dev6.  
  - Edge Cases: Missing inputs, merge conflicts.

- **consolidator**  
  - Type: Code (JavaScript)  
  - Role: Consolidates fragmented JSON workflow strings into a single string separated by a defined fragment separator. Also extracts metadata for validation.  
  - Input: Merged fragments from merge_all_fragments2.  
  - Output: Consolidated string with metadata: signatures found, nodes count, ready flags.  
  - Edge Cases: JSON parsing errors, empty inputs.

---

### 1.5 Dynamic Connection Creation

**Overview:**  
This block refactors node IDs to eliminate duplicates, intelligently classifies nodes by function, and creates logical connections to form a valid n8n workflow without loops.

**Nodes Involved:**  
- Code3 (Code node)  

#### Node Details:

- **Code3**  
  - Type: Code (JavaScript)  
  - Role:  
    - Parses consolidated workflow fragments.  
    - Detects and fixes duplicate node IDs by assigning unique IDs.  
    - Classifies nodes into categories: triggers, processors, routers, actions, outputs.  
    - Creates connections:  
      - Triggers → Processors (entry point)  
      - Processors chained sequentially and then to Router (if any)  
      - Router branches to Actions  
      - Actions converge to Outputs or chain sequentially if no output  
    - Returns detailed metadata and connection map.  
  - Edge Cases: Malformed JSON, missing fields, logic errors creating loops.

---

### 1.6 Workflow Sending

**Overview:**  
This block sends the finalized workflow and message to the recipient’s Magic Inbox or Telegram channel.

**Nodes Involved:**  
- Message server n8n send (Magic Inbox Send)  
- 📤 Magic Inbox Send P2P (Magic Inbox Send)  

#### Node Details:

- **Message server n8n send**  
  - Type: Magic Inbox Send  
  - Role: Sends a message with a premium workflow payload to recipients via Magic Inbox.  
  - Parameters:  
    - Message text (example thank you note).  
    - Sender ID with Magic Inbox URL.  
    - Address book with recipient Magic Inbox URLs.  
    - Content type: "workflow_with_message" including a JSON workflow embedded.  
  - Edge Cases: Invalid Magic Inbox URLs, API failures.

- **📤 Magic Inbox Send P2P**  
  - Type: Magic Inbox Send  
  - Role: Similar to above, configured for sending workflows P2P with example addresses and messages.  
  - Edge Cases: Same as above.

---

## 3. Summary Table

| Node Name                   | Node Type                            | Functional Role                              | Input Node(s)                      | Output Node(s)                     | Sticky Note                                            |
|-----------------------------|------------------------------------|----------------------------------------------|-----------------------------------|----------------------------------|--------------------------------------------------------|
| Message server n8n trigger   | Telegram Trigger                   | Receive Telegram messages as workflow triggers | —                                 | Message server n8n send           |                                                        |
| Message server n8n send      | Magic Inbox Send                  | Send message/workflow via Magic Inbox         | Message server n8n trigger         | —                                | 🟩 For sending P2P independent workflow                |
| Receiving message server n8n trigger | Magic Inbox Trigger             | Receive Magic Inbox P2P messages               | —                                 | Receiving message server n8n      | 🟦 For receiving P2P independent workflow              |
| Receiving message server n8n | Telegram                          | Send formatted Telegram messages               | Receiving message server n8n trigger | —                                |                                                        |
| When clicking               | Manual Trigger                   | Manual initiation for testing                   | —                                 | 📤 Magic Inbox Send P2P           |                                                        |
| timestamp_generator1         | DateTime                         | Generate timestamp for tracking                 | When chat message received         | team_leader_dispatcher1           | TIMESTAMP: Generates current timestamp for workflow naming and tracking. Configured for Paris timezone to match user location. |
| team_leader_dispatcher1      | Perplexity AI                    | Analyze user request, distribute tasks          | timestamp_generator1               | information_extractor1            | TEAM LEADER: Strategic analysis and task distribution using Perplexity AI. |
| information_extractor1       | Langchain Information Extractor | Extract structured JSON from dispatcher output  | team_leader_dispatcher1            | golden_dev, golden_dev1,...       | EXTRACTION: Extracts structured data from dispatcher analysis using LangChain. Disabled. |
| OpenAI Chat Model5           | Langchain OpenAI Chat Model      | Support information extractor                   | —                                 | information_extractor1            |                                                        |
| golden_dev                  | Langchain Agent                  | Coordination AI agent (merge, switch, if, wait) | Think Visionnaire1                 | merge_all_fragments2              |                                                        |
| golden_dev1                 | Langchain Agent                  | Triggering AI agent (webhook, schedule, manual) | Think Visionnaire2                 | merge_all_fragments2              |                                                        |
| golden_dev2                 | Langchain Agent                  | Logic AI agent (function, code, date-time)      | Think Visionnaire3                 | merge_all_fragments2              |                                                        |
| golden_dev3                 | Langchain Agent                  | Integration AI agent (HTTP request, OAuth)       | Think Visionnaire4                 | merge_all_fragments2              |                                                        |
| golden_dev4                 | Langchain Agent                  | Data AI agent (set, respond to webhook)          | Think Visionnaire7                 | merge_all_fragments2              |                                                        |
| golden_dev5                 | Langchain Agent                  | Communication AI agent (email, slack, discord)   | Think Visionnaire5                 | merge_all_fragments2              |                                                        |
| golden_dev6                 | Langchain Agent                  | Robustness AI agent (error handling, monitoring) | Think Visionnaire6                 | merge_all_fragments2              |                                                        |
| Think Visionnaire1 to 7      | toolThink                       | Intermediate prompt processing nodes             | team_leader_dispatcher1 outputs   | golden_dev agents                |                                                        |
| OpenRouter Chat Model to Chat Model6 | Langchain OpenRouter Nodes    | AI language models backing golden_dev agents     | Think Visionnaire nodes            | golden_dev agents                |                                                        |
| merge_all_fragments2         | Merge                           | Combine all AI agent outputs into unified fragment | golden_dev, golden_dev1,...        | consolidator                    | FRAGMENT MERGER: Combines all specialist outputs into unified structure. |
| consolidator                | Code                            | Consolidate fragments into a single workflow string | merge_all_fragments2               | Code3                           | CONSOLIDATEUR UNIVERSEL V2 - MAGIC DEV GOLDEN           |
| Code3                       | Code                            | Refactor node IDs, classify nodes, create connections | consolidator                     | —                              | CONNECTEUR ULTRA-DYNAMIQUE V4 - CODE COMPLET CORRIGÉ     |
| 📤 Magic Inbox Send P2P       | Magic Inbox Send                | Send workflow and message via Magic Inbox P2P    | When clicking                    | —                              | 🟩 For sending P2P independent workflow                  |
| 📬 Magic Inbox P2P            | Magic Inbox                    | Receive workflows/messages via Magic Inbox P2P   | —                                 | Receiving message server n8n      |                                                        |

---

## 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger Node**  
   - **Type:** n8n-nodes-base.telegramTrigger  
   - **Name:** Message server n8n trigger  
   - **Configuration:**  
     - Updates: message  
     - Credentials: Telegram API with your group/channel API  
   - Connect to Message server n8n send node.

2. **Create Magic Inbox Trigger Node**  
   - **Type:** n8n-nodes-magic-inbox.magicInbox  
   - **Name:** Receiving message server n8n trigger  
   - **Parameters:**  
     - n8nApiKey: Your n8n API Key  
     - n8nInstanceUrl: Your n8n instance URL  
   - Connect to Receiving message server n8n node.

3. **Create Telegram Send Node**  
   - **Type:** n8n-nodes-base.telegram  
   - **Name:** Receiving message server n8n  
   - **Parameters:**  
     - Chat ID: Target Telegram chat  
     - Text: Format message with input JSON fields (`from`, `message`, `workflowImport.message`, `receivedAt`)  
   - Credentials: Telegram API for sending.

4. **Create Manual Trigger Node**  
   - **Type:** n8n-nodes-base.manualTrigger  
   - **Name:** When clicking  
   - For manual testing and initiating P2P sends.

5. **Create Magic Inbox Send Node**  
   - **Type:** n8n-nodes-magic-inbox.magicInboxSend  
   - **Name:** 📤 Magic Inbox Send P2P  
   - **Parameters:**  
     - Message: Example thank you text  
     - SenderId: Your name and Magic Inbox URL  
     - AddressBook: List recipients’ Magic Inbox URLs  
     - ContentType: workflow_with_message  
     - WorkflowJson: Embed JSON of workflow to send  

6. **Create Timestamp Generator Node**  
   - **Type:** n8n-nodes-base.dateTime  
   - **Name:** timestamp_generator1  
   - **Parameters:**  
     - Output field name: timestamp  
     - Timezone: Europe/Paris  

7. **Create Team Leader Dispatcher Node**  
   - **Type:** n8n-nodes-base.perplexity  
   - **Name:** team_leader_dispatcher1  
   - **Parameters:**  
     - Model: sonar-reasoning-pro  
     - Messages: System prompt for detailed task distribution, user input via expression  
   - Connect output of timestamp_generator1 to this node.

8. **Create Specialized AI Agent Nodes**  
   - For each golden_dev, golden_dev1 to golden_dev6:  
     - Type: @n8n/n8n-nodes-langchain.agent  
     - Configure promptType as "define" with instructions from dispatcher output for each specialization.  
     - Connect each to respective Think Visionnaire toolThink nodes and OpenRouter Chat Models.  
   - Use Anthropic Claude-sonnet-4 model via OpenRouter API with credentials.

9. **Create Think Visionnaire Nodes**  
   - Type: @n8n/n8n-nodes-langchain.toolThink  
   - Name: Think Visionnaire1 to Think Visionnaire7  
   - Connect dispatcher output to these nodes as AI tools feeding respective golden_dev agents.

10. **Create Merge Node**  
    - Type: n8n-nodes-base.merge  
    - Name: merge_all_fragments2  
    - Mode: combine by position  
    - Number Inputs: 7  
    - Connect all golden_dev agents’ outputs to the merge node inputs.

11. **Create Consolidator Code Node**  
    - Type: n8n-nodes-base.code  
    - Name: consolidator  
    - Paste JS code to consolidate workflow fragments into a single JSON string with metadata.

12. **Create Code3 Node for Dynamic Connections**  
    - Type: n8n-nodes-base.code  
    - Name: Code3  
    - Paste JS code to:  
      - Fix duplicate IDs  
      - Classify nodes  
      - Create logical connections (trigger → processor → router → actions → output)  
    - Connect consolidator output to Code3 input.

13. **Connect Final Output**  
    - Use Code3 output for further sending or deployment.

14. **Set Credentials**  
    - Telegram API credentials for trigger and send nodes.  
    - Magic Inbox API keys and URLs for Magic Inbox nodes.  
    - OpenRouter API credentials for Langchain AI nodes.  
    - OpenAI API credentials for OpenAI Chat Model.  

15. **Create Sticky Notes**  
    - Optional: Add sticky notes in n8n to annotate sections such as "Magic Inbox Hub", "Features", "Modular Design", "P2P Send/Receive" for clarity.

---

## 5. General Notes & Resources

| Note Content                                                                                                      | Context or Link                                                                                   |
|------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| 🟩 Magic Inbox: A hub for automating peer-to-peer workflow sharing between n8n instances.                        | Sticky Note content in workflow.                                                                |
| ✨ System features include server-to-server Telegram channels, direct chat/workflow sharing, and P2P workflow movement. | Sticky Note content describing system capabilities.                                             |
| 🟨 Modular design allows creating Telegram groups with server-to-server communication.                           | Sticky Note suggests creating Telegram groups like "Telegram server address" for clarity.       |
| 🟩 Use Magic Inbox Send nodes to send independent workflows P2P with example sender and address book settings.   | Sticky Note explains creating P2P sending addresses.                                           |
| 🟦 Use Magic Inbox triggers to receive independent workflows P2P with example receiving addresses.               | Sticky Note explains creating P2P receiving addresses.                                         |
| Workflow follows "Golden Dev" architecture assigning specialized roles to nodes: coordination, triggering, logic, integration, data, communication, robustness. | System prompt and node naming conventions inside AI agent prompts.                              |
| AI agents use Anthropic Claude-sonnet-4 model via OpenRouter API for specialized workflow code generation.       | Requires OpenRouter API account and credentials setup.                                          |
| Dispatcher uses Perplexity AI to analyze user requests and distribute tasks.                                     | Requires Perplexity API and configured credentials in n8n.                                     |
| Code nodes contain detailed JavaScript for consolidating workflow fragments and constructing logical node connections. | Core logic for final workflow assembly and validation.                                         |
| Workflow timezone is set to Europe/Paris consistently for timestamping and scheduling.                           | Workflow settings in multiple nodes.                                                           |

---

# DISCLAIMER

The provided content comes exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.

---

# End of Document