Save Telegram Text, Voice & Audio to Notion with DeepSeek & OpenAI Summaries

https://n8nworkflows.xyz/workflows/save-telegram-text--voice---audio-to-notion-with-deepseek---openai-summaries-4188


# Save Telegram Text, Voice & Audio to Notion with DeepSeek & OpenAI Summaries

---

### 1. Workflow Overview

This workflow automates the processing and storage of Telegram messages—specifically text, voice notes, and audio files—by saving their content to a Notion database. It leverages DeepSeek’s AI chat models combined with OpenAI summarization to generate meaningful summaries or transcriptions of audio content. The workflow is designed for personal data reporting and note-taking use cases, enabling users to archive Telegram communications with AI-enhanced summaries in Notion.

The workflow logic is organized into the following blocks:

- **1.1 Input Reception:** Listens for incoming Telegram messages (text, voice, or audio) via Telegram triggers.
- **1.2 Message Type Routing:** Uses a Switch node to classify messages and route them based on content type.
- **1.3 AI Processing Chains:** Processes messages through one of several AI pipelines using DeepSeek chat models and OpenAI nodes to create summaries or transcriptions.
- **1.4 Data Storage:** Stores processed content and metadata in various Notion databases.
- **1.5 AI Agent Coordination:** Coordinates AI models with agent nodes to manage complex AI workflows.
- **1.6 Auxiliary Telegram Nodes:** Additional Telegram nodes handle message retrieval or further interaction downstream when required.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:** This block monitors Telegram for new incoming messages and starts the workflow upon reception.
- **Nodes Involved:**  
  - Telegram Trigger
- **Node Details:**
  - **Telegram Trigger**
    - Type: Trigger node listening for Telegram updates.
    - Configuration: Default settings with a webhook ID; listens for all message types.
    - Inputs: None (starts workflow).
    - Outputs: Feeds data to the Switch node.
    - Edge cases: Telegram API downtime, webhook misconfiguration.
    - Version: 1
    - Sticky Note: None

#### 2.2 Message Type Routing

- **Overview:** Routes incoming Telegram messages based on their type (text, voice, audio) to appropriate AI processing paths.
- **Nodes Involved:**  
  - Switch1  
  - Telegram  
  - Telegram1
- **Node Details:**
  - **Switch1**
    - Type: Decision node to route messages based on their content type.
    - Configuration: Conditions to check message type (e.g., text, voice note, or audio file).
    - Inputs: From Telegram Trigger.
    - Outputs: Three outputs connected to Telegram, AI Agent, and Telegram1 nodes.
    - Edge cases: Messages without recognizable content type, unexpected message formats.
    - Version: 3.2
  - **Telegram**
    - Type: Telegram node (regular).
    - Configuration: Used to retrieve or handle specific message types routed from Switch1.
    - Inputs: From Switch1.
    - Outputs: Connected to an OpenAI node.
    - Edge cases: Telegram API errors, message not found.
    - Version: 1.2
  - **Telegram1**
    - Type: Telegram node (secondary).
    - Configuration: Similar role as Telegram node but for a different message branch.
    - Inputs: From Switch1.
    - Outputs: Connected to OpenAI2 node.
    - Edge cases: Same as Telegram node.
    - Version: 1.2
  - Sticky Notes: None

#### 2.3 AI Processing Chains

- **Overview:** Uses AI language models (DeepSeek and OpenAI) and AI agent nodes to process text or audio data into summaries or enriched content.
- **Nodes Involved:**  
  - DeepSeek Chat Model  
  - AI Agent  
  - OpenAI  
  - DeepSeek Chat Model1  
  - AI Agent1  
  - OpenAI2  
  - DeepSeek Chat Model2  
  - AI Agent2
- **Node Details:**

  - **DeepSeek Chat Model**
    - Type: AI language model node using DeepSeek.
    - Configuration: Default or custom DeepSeek chat model parameters.
    - Inputs: None directly; linked as language model for AI Agent.
    - Outputs: Feeds AI Agent.
    - Edge cases: Model API authentication, response delays.
    - Version: 1
  - **AI Agent**
    - Type: LangChain Agent node coordinating AI models.
    - Configuration: Uses DeepSeek Chat Model as language model.
    - Inputs: From OpenAI or other nodes.
    - Outputs: Connects to Notion1.
    - Edge cases: Agent logic failures, misconfiguration.
    - Version: 1.9
  - **OpenAI**
    - Type: OpenAI node for text summarization or processing.
    - Configuration: Connected downstream of Telegram node.
    - Inputs: From Telegram.
    - Outputs: Feeds AI Agent2.
    - Credentials: Requires valid OpenAI API key.
    - Edge cases: API rate limits, malformed prompts.
    - Version: 1
  - **DeepSeek Chat Model1**
    - Type: Secondary DeepSeek chat model node.
    - Configuration: Similar to primary DeepSeek node but used in alternate branch.
    - Inputs: Feeds AI Agent1.
    - Outputs: Connected to AI Agent1.
    - Edge cases: Same as DeepSeek Chat Model.
    - Version: 1
  - **AI Agent1**
    - Type: Agent node for managing AI flow on alternate branch.
    - Inputs: From OpenAI2.
    - Outputs: Connects to Notion3.
    - Version: 1.9
  - **OpenAI2**
    - Type: Secondary OpenAI node for summarization.
    - Inputs: From Telegram1.
    - Outputs: Feeds AI Agent1.
    - Credentials: Requires OpenAI API key.
    - Version: 1
  - **DeepSeek Chat Model2**
    - Type: Additional DeepSeek chat model instance.
    - Inputs: Feeds AI Agent2.
    - Outputs: Connects to AI Agent2.
    - Version: 1
  - **AI Agent2**
    - Type: Agent node managing AI logic for third processing path.
    - Inputs: From OpenAI.
    - Outputs: Connects to Notion4.
    - Version: 1.9
  - Sticky Notes:
    - Sticky Note, Sticky Note3: Positioned around OpenAI2 and AI Agent1 nodes.
    - Sticky Note4: Near Switch1, possibly indicating message routing logic.

#### 2.4 Data Storage

- **Overview:** Saves processed data (summaries, transcripts, metadata) into various Notion databases.
- **Nodes Involved:**  
  - Notion1  
  - Notion2  
  - Notion3  
  - Notion4  
  - Notion
- **Node Details:**
  - **Notion1**
    - Type: Notion node for database item creation or update.
    - Configuration: Connected to AI Agent output.
    - Inputs: From AI Agent.
    - Outputs: None.
    - Credentials: Requires Notion API authorization.
    - Edge cases: API limits, database schema mismatch.
    - Version: 2.2
  - **Notion2**
    - Type: Notion node connected to AI Agent1 output.
    - Inputs: From AI Agent1.
    - Outputs: None.
    - Version: 2.1
  - **Notion3**
    - Type: Notion node connected to AI Agent1.
    - Inputs: From AI Agent1.
    - Outputs: None.
    - Version: 2.2
  - **Notion4**
    - Type: Notion node connected to AI Agent2.
    - Inputs: From AI Agent2.
    - Outputs: None.
    - Version: 2.2
  - **Notion**
    - Type: Notion node downstream of OpenAI.
    - Inputs: From OpenAI.
    - Outputs: None.
    - Version: 2.1
  - Sticky Notes:
    - Sticky Note2: Near Notion and Notion2 nodes, possibly indicating database setup or instructions.

#### 2.5 Auxiliary Telegram Nodes

- **Overview:** Additional Telegram nodes used for message retrieval or further Telegram interaction depending on routing.
- **Nodes Involved:**  
  - Telegram  
  - Telegram1
- **Node Details:**
  - See section 2.2 for details as these nodes act in message-type routing.

---

### 3. Summary Table

| Node Name           | Node Type                           | Functional Role                | Input Node(s)       | Output Node(s)          | Sticky Note                                                      |
|---------------------|-----------------------------------|-------------------------------|---------------------|-------------------------|-----------------------------------------------------------------|
| Telegram Trigger     | telegramTrigger                   | Start trigger for Telegram msgs | None                | Switch1                 |                                                                 |
| Switch1             | switch                           | Routes messages by type        | Telegram Trigger    | Telegram, AI Agent, Telegram1 | Sticky Note4 near Switch1 indicates routing logic               |
| Telegram            | telegram                         | Handles Telegram messages      | Switch1             | OpenAI                  |                                                                 |
| Telegram1           | telegram                         | Secondary Telegram handler     | Switch1             | OpenAI2                 |                                                                 |
| OpenAI              | openAi                          | Text summarization             | Telegram             | AI Agent2               |                                                                 |
| OpenAI2             | openAi                          | Secondary text summarization   | Telegram1            | AI Agent1               | Sticky Note, Sticky Note3 near OpenAI2 and AI Agent1            |
| DeepSeek Chat Model  | lmChatDeepSeek                  | AI language model (DeepSeek)  | None (LM for AI Agent) | AI Agent                |                                                                 |
| DeepSeek Chat Model1 | lmChatDeepSeek                  | Secondary DeepSeek LM          | None (LM for AI Agent1) | AI Agent1               |                                                                 |
| DeepSeek Chat Model2 | lmChatDeepSeek                  | Additional DeepSeek LM         | None (LM for AI Agent2) | AI Agent2               |                                                                 |
| AI Agent            | agent                           | Coordinates AI processing      | OpenAI               | Notion1                 |                                                                 |
| AI Agent1           | agent                           | Coordinates AI processing      | OpenAI2              | Notion3                 |                                                                 |
| AI Agent2           | agent                           | Coordinates AI processing      | OpenAI               | Notion4                 |                                                                 |
| Notion1             | notion                          | Saves processed data           | AI Agent             | None                    |                                                                 |
| Notion2             | notion                          | Saves processed data           | AI Agent1             | None                    | Sticky Note2 near Notion nodes possibly about database usage    |
| Notion3             | notion                          | Saves processed data           | AI Agent1             | None                    |                                                                 |
| Notion4             | notion                          | Saves processed data           | AI Agent2             | None                    |                                                                 |
| Notion              | notion                          | Saves processed data           | OpenAI                | None                    |                                                                 |
| Sticky Note1        | stickyNote                      | Empty or no content            | None                 | None                    |                                                                 |
| Sticky Note2        | stickyNote                      | Possibly database instructions| None                 | None                    | Near Notion nodes                                                |
| Sticky Note3        | stickyNote                      | Possibly AI workflow notes     | None                 | None                    | Near OpenAI2 and AI Agent1                                      |
| Sticky Note4        | stickyNote                      | Routing logic notes            | None                 | None                    | Near Switch1                                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger Node:**
   - Set node type to “Telegram Trigger”.
   - Configure webhook with Telegram bot credentials.
   - Enable listening to all message types.
   - Position as the starting node.

2. **Add Switch Node (Switch1):**
   - Add a “Switch” node after Telegram Trigger.
   - Configure conditions to check message type:
     - Output 1: Text messages → connect to Telegram node.
     - Output 2: Voice notes → connect to AI Agent node.
     - Output 3: Audio files → connect to Telegram1 node.

3. **Create Telegram Nodes (Telegram and Telegram1):**
   - For each, set node type to “Telegram”.
   - Configure with same Telegram credentials.
   - Connect Telegram to output 1 of Switch1.
   - Connect Telegram1 to output 3 of Switch1.

4. **Add OpenAI Nodes (OpenAI and OpenAI2):**
   - Create two OpenAI nodes configured with valid OpenAI API credentials.
   - Connect OpenAI to output of Telegram node.
   - Connect OpenAI2 to output of Telegram1 node.

5. **Add DeepSeek Chat Model Nodes (DeepSeek Chat Model, DeepSeek Chat Model1, DeepSeek Chat Model2):**
   - Create three LangChain DeepSeek chat model nodes.
   - Configure each with appropriate DeepSeek credentials and parameters.
   - For each, set them as language models for respective AI Agent nodes.

6. **Add AI Agent Nodes (AI Agent, AI Agent1, AI Agent2):**
   - Create three AI Agent nodes configured to use corresponding DeepSeek chat model nodes as language models.
   - Connect OpenAI node output to AI Agent2.
   - Connect OpenAI2 node output to AI Agent1.
   - Connect AI Agent node output from Switch1 output 2.
   - Connect AI Agent nodes outputs to respective Notion nodes.

7. **Add Notion Nodes (Notion, Notion1, Notion2, Notion3, Notion4):**
   - Create five Notion nodes.
   - Configure each with Notion API credentials and target database.
   - Connect AI Agent outputs to these Notion nodes accordingly:
     - AI Agent → Notion1
     - AI Agent1 → Notion3
     - AI Agent2 → Notion4
   - Connect OpenAI output to Notion node.
   - Connect AI Agent1 output to Notion2 as well (if applicable).

8. **Connect Nodes:**
   - Telegram Trigger → Switch1
   - Switch1 outputs → Telegram / AI Agent / Telegram1
   - Telegram → OpenAI → AI Agent2 → Notion4
   - Telegram1 → OpenAI2 → AI Agent1 → Notion3
   - AI Agent (from Switch1 output 2) → Notion1
   - OpenAI → Notion (direct)

9. **Credentials Setup:**
   - Setup Telegram Bot credentials for Telegram Trigger and Telegram nodes.
   - Setup OpenAI API key credentials for OpenAI and OpenAI2 nodes.
   - Setup DeepSeek API credentials for DeepSeek Chat Model nodes.
   - Setup Notion API credentials with access to target databases for Notion nodes.

10. **Optional: Add Sticky Notes:**
    - Add sticky notes near Switch1 explaining routing logic.
    - Add sticky notes near Notion nodes for database instructions.
    - Add sticky notes near OpenAI2 and AI Agent1 nodes explaining AI workflow details.

---

### 5. General Notes & Resources

| Note Content                                                                                                    | Context or Link                                                                                                   |
|----------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------|
| This workflow uses DeepSeek chat models integrated via LangChain nodes for enhanced AI text/audio processing.  | n8n LangChain DeepSeek documentation (internal node type `@n8n/n8n-nodes-langchain.lmChatDeepSeek`)              |
| OpenAI nodes require an API key with access to GPT models for summarization and text generation.               | OpenAI official docs: https://platform.openai.com/docs/                                                           |
| Telegram Trigger and Telegram nodes require a Telegram Bot with webhook properly configured in Telegram Bot API.| Telegram Bot API documentation: https://core.telegram.org/bots/api                                                |
| Notion nodes need integration tokens with permissions to write to target Notion databases.                     | Notion API docs: https://developers.notion.com/docs/getting-started                                                |
| The Switch node is critical for routing messages by type—ensure message type detection logic matches Telegram payloads.|                                                                                                                  |

---

Disclaimer: The provided text is sourced exclusively from an automated workflow created with n8n, a tool for integration and automation. The processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All processed data are legal and public.

---