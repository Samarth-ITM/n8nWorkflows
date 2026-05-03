Generate Cold Emails & Sales Letters with OpenAI GPT & Google Docs via Chat

https://n8nworkflows.xyz/workflows/generate-cold-emails---sales-letters-with-openai-gpt---google-docs-via-chat-5828


# Generate Cold Emails & Sales Letters with OpenAI GPT & Google Docs via Chat

---

### 1. Workflow Overview

**Purpose:**  
This workflow automates the generation of cold emails, sales letters, and other marketing copy using OpenAI GPT models integrated with Google Docs. It serves as a centralized AI-powered copywriting platform that receives chat input, identifies the type of marketing content requested, routes the request to specialized AI agents, and then delivers and archives the final copy in a shared Google Document.

**Target Use Cases:**  
- Generating cold email sequences for outreach campaigns  
- Drafting long-form sales letters with storytelling and persuasive elements  
- Producing various types of marketing copy via specialized AI agents  
- Maintaining conversational context for follow-up interactions  
- Automatically updating shared Google Docs with finalized content

**Logical Blocks:**  
- **1.1 Input Reception:** Captures user chat requests via a chat trigger node.  
- **1.2 AI Processing & Routing:** Uses a control agent node to interpret user intent and route requests to specialized copywriting workflows or models.  
- **1.3 AI Language Model Interaction:** Engages an OpenAI GPT-4 model for text generation as part of the processing pipeline.  
- **1.4 Conversational Memory:** Maintains context across interactions to enable coherent multi-turn conversations.  
- **1.5 Output Delivery & Documentation:** Sends the generated copy back to the user and updates a Google Docs document with the final text.  
- **1.6 Specialized Agent Workflows:** Invokes sub-workflows dedicated to cold email writing and sales letter creation.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  Receives incoming chat messages from users to trigger the workflow.

- **Nodes Involved:**  
  - When chat message received

- **Node Details:**  
  - **When chat message received**  
    - *Type:* LangChain Chat Trigger  
    - *Role:* Initiates the workflow on receiving a chat message from an external source (e.g., chat interface or platform).  
    - *Configuration:* Default trigger with webhook enabled. No additional options specified.  
    - *Input:* Webhook trigger from chat message.  
    - *Output:* Passes chat message data to the Copy Assistant node.  
    - *Edge Cases:* Potential webhook failures or missing message content could disrupt triggering.  
    - *Version:* 1.1  

#### 1.2 AI Processing & Routing

- **Overview:**  
  Acts as the central control agent determining user intent and routing the request to the appropriate specialized copywriting tool or sub-workflow.

- **Nodes Involved:**  
  - Copy Assistant

- **Node Details:**  
  - **Copy Assistant**  
    - *Type:* LangChain Agent Node  
    - *Role:* Receives input, interprets marketing copy requests, and dispatches them to specialized AI tools or workflows (e.g., Cold Email Copywriter, Sales Letter Agent). Coordinates integration with memory and language model nodes.  
    - *Configuration:*  
      - Uses a detailed system prompt describing role, context, tools, SOPs, and examples for intent interpretation and routing.  
      - Routes requests based on keywords and intent to:  
        - Sales Letter Agent (long-form sales letters)  
        - Cold Email Copywriter (email sequences)  
        - Other agents (Ad Copy, VSL Script, Ad Script) mentioned in prompt but not active in this workflow.  
      - Manages the final delivery of copy and updates the Google Doc.  
    - *Expressions:* Uses the prompt to map user input to agents; no explicit variable expressions beyond standard LangChain prompt usage.  
    - *Input:* Receives user chat message from "When chat message received" node and AI model output from OpenAI Chat Model.  
    - *Output:* Sends processed copy to "Update a document" node and calls relevant tool workflows.  
    - *Edge Cases:*  
      - Incorrect intent detection may route to wrong agent.  
      - Agent failure or sub-workflow invocation errors.  
      - Dependency on memory and AI model availability.  
    - *Version:* 2  

#### 1.3 AI Language Model Interaction

- **Overview:**  
  Provides core text generation capabilities using OpenAI GPT-4, supporting the Copy Assistant’s processing and generation tasks.

- **Nodes Involved:**  
  - OpenAI Chat Model

- **Node Details:**  
  - **OpenAI Chat Model**  
    - *Type:* LangChain OpenAI Chat Model Node  
    - *Role:* Generates AI language responses based on prompts from the Copy Assistant agent.  
    - *Configuration:*  
      - Model set to "gpt-4.1-mini" for advanced conversational AI.  
      - No additional options enabled.  
    - *Credentials:* Uses OpenAI API credentials securely stored in n8n.  
    - *Input:* Receives prompt from Copy Assistant.  
    - *Output:* Returns generated text back to Copy Assistant.  
    - *Edge Cases:* API rate limits, network errors, or invalid prompts can cause failures or degraded responses.  
    - *Version:* 1.2  

#### 1.4 Conversational Memory

- **Overview:**  
  Maintains a sliding window buffer of conversational context to allow the Copy Assistant to generate contextually relevant and coherent responses.

- **Nodes Involved:**  
  - Simple Memory

- **Node Details:**  
  - **Simple Memory**  
    - *Type:* LangChain Memory Buffer Window Node  
    - *Role:* Stores recent conversation text to provide context to the AI agent during multi-turn interactions.  
    - *Configuration:* Default buffer window memory with no custom parameters specified.  
    - *Input:* Receives conversation data from Copy Assistant.  
    - *Output:* Provides stored context back to Copy Assistant for prompt enrichment.  
    - *Edge Cases:* Memory overflow or loss may reduce context quality; no explicit persistence beyond session.  
    - *Version:* 1.3  

#### 1.5 Output Delivery & Documentation

- **Overview:**  
  Delivers the final generated copy to the user and updates a shared Google Docs document with the content.

- **Nodes Involved:**  
  - Update a document

- **Node Details:**  
  - **Update a document**  
    - *Type:* Google Docs Node  
    - *Role:* Inserts the generated marketing copy text into a specified Google Docs document for collaborative editing and record keeping.  
    - *Configuration:*  
      - Operation set to "update" with action to insert the generated text at the current cursor or end of document.  
      - Document URL points to a specific Google Docs file shared across the team.  
    - *Credentials:* Uses OAuth2 credentials for Google Docs API access.  
    - *Input:* Receives finalized text output from Copy Assistant.  
    - *Output:* None (terminal node in this flow).  
    - *Edge Cases:*  
      - OAuth token expiry or insufficient permissions may block updates.  
      - Document access errors if URL or permissions change.  
    - *Version:* 2  

#### 1.6 Specialized Agent Workflows

- **Overview:**  
  External workflows dedicated to generating specific types of marketing copy (cold emails and sales letters) are called as tools by the Copy Assistant.

- **Nodes Involved:**  
  - Cold Email Writer Tool  
  - Sales Letter Tool

- **Node Details:**  
  - **Cold Email Writer Tool**  
    - *Type:* LangChain Tool Workflow Node  
    - *Role:* Invokes a sub-workflow specialized in creating cold email sequences.  
    - *Configuration:*  
      - References workflow ID "ZtScEU9B5PNPnIcP" (Cold Email Copywriter).  
      - No input parameters mapped explicitly, designed to receive context and user brief via the main workflow.  
    - *Input:* Called by Copy Assistant upon intent detection.  
    - *Output:* Returns generated cold email copy to Copy Assistant.  
    - *Edge Cases:* Sub-workflow errors or misconfiguration may cause generation failure.  
    - *Version:* 2.2  

  - **Sales Letter Tool**  
    - *Type:* LangChain Tool Workflow Node  
    - *Role:* Invokes a sub-workflow specialized in creating long-form sales letters.  
    - *Configuration:*  
      - References workflow ID "HQuj3iBV1PlI0HY1" (Sales Letter Agent).  
      - Similar input/output behavior as Cold Email tool.  
    - *Input:* Called by Copy Assistant upon intent detection.  
    - *Output:* Returns generated sales letter copy to Copy Assistant.  
    - *Edge Cases:* Same as above regarding sub-workflow availability and correctness.  
    - *Version:* 2.2  

---

### 3. Summary Table

| Node Name               | Node Type                     | Functional Role                            | Input Node(s)           | Output Node(s)          | Sticky Note                               |
|-------------------------|-------------------------------|-------------------------------------------|-------------------------|-------------------------|-------------------------------------------|
| When chat message received | LangChain Chat Trigger       | Entry point, receives user chat messages | —                       | Copy Assistant          |                                           |
| Copy Assistant          | LangChain Agent               | Intent interpretation, routing, delivery | When chat message received, OpenAI Chat Model, Cold Email Writer Tool, Sales Letter Tool, Simple Memory | Update a document       |                                           |
| OpenAI Chat Model       | LangChain OpenAI Chat Model   | Generates AI text for Copy Assistant      | Copy Assistant          | Copy Assistant          |                                           |
| Simple Memory           | LangChain Memory Buffer Window| Maintains conversational context          | Copy Assistant          | Copy Assistant          |                                           |
| Update a document       | Google Docs Node              | Updates shared Google Doc with final copy | Copy Assistant          | —                       |                                           |
| Cold Email Writer Tool  | LangChain Tool Workflow       | Generates cold email sequences             | Copy Assistant          | Copy Assistant          |                                           |
| Sales Letter Tool       | LangChain Tool Workflow       | Generates sales letters                     | Copy Assistant          | Copy Assistant          |                                           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Chat Trigger Node (When chat message received):**  
   - Type: LangChain Chat Trigger  
   - Configure to listen for incoming chat messages.  
   - Enable webhook and note the webhook URL for receiving messages.

2. **Add the Copy Assistant Agent Node:**  
   - Type: LangChain Agent  
   - Paste the system message prompt defining roles, context, tools, SOPs, and examples for interpreting copywriting requests.  
   - Configure connections for AI language model, memory, and tool workflows (Cold Email Writer and Sales Letter tools).  
   - Connect the output of the Chat Trigger to this node's main input.

3. **Configure OpenAI Chat Model Node:**  
   - Type: LangChain OpenAI Chat Model  
   - Select model "gpt-4.1-mini" or equivalent GPT-4 variant.  
   - Link OpenAI API credentials via OAuth or API key.  
   - Connect this node's output to the Copy Assistant’s AI language model input.

4. **Set up Simple Memory Node:**  
   - Type: LangChain Memory Buffer Window  
   - Use default buffer window memory to maintain conversational context.  
   - Connect input/output to/from Copy Assistant node.

5. **Configure Google Docs Update Node:**  
   - Type: Google Docs Node  
   - Operation: Update document  
   - Action: Insert text  
   - Document URL: Paste your shared Google Docs URL where final copy should be inserted.  
   - Authenticate with Google Docs OAuth2 credentials.  
   - Connect output of Copy Assistant node to this node.

6. **Add Cold Email Writer Tool Node:**  
   - Type: LangChain Tool Workflow Node  
   - Set workflow ID to the dedicated cold email sub-workflow.  
   - No special inputs are defined; it receives data routed from Copy Assistant.  
   - Connect output to Copy Assistant’s tool input.

7. **Add Sales Letter Tool Node:**  
   - Type: LangChain Tool Workflow Node  
   - Set workflow ID to the dedicated sales letter sub-workflow.  
   - Connect output to Copy Assistant’s tool input.

8. **Wire Connections:**  
   - Chat Trigger → Copy Assistant (main)  
   - Copy Assistant → Update a document  
   - Copy Assistant ↔ OpenAI Chat Model (ai_languageModel)  
   - Copy Assistant ↔ Simple Memory (ai_memory)  
   - Copy Assistant ↔ Cold Email Writer Tool (ai_tool)  
   - Copy Assistant ↔ Sales Letter Tool (ai_tool)  

9. **Set Credentials:**  
   - OpenAI API credentials for the OpenAI Chat Model node.  
   - Google OAuth2 API credentials for Google Docs node.

10. **Test Workflow:**  
    - Send test chat messages simulating requests for cold emails or sales letters.  
    - Confirm that the Copy Assistant routes requests properly, generates copy, and updates Google Docs.

---

### 5. General Notes & Resources

| Note Content                                                                                      | Context or Link                                                                                                 |
|-------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------|
| Workflow integrates multiple specialized copywriting agents via a central AI agent coordinator. | Supports scalable and modular marketing copy creation workflows.                                              |
| Uses OpenAI GPT-4 “mini” chat model for efficient conversational AI generation.                 | Ensures advanced language capabilities with lower resource consumption.                                        |
| Google Docs integration enables real-time collaborative editing and archiving of generated copy. | https://docs.google.com/document/d/1eI4Pdje4As0KEYjEG3prMUQCiLe-tUswnwNUVQG2GSw/edit?tab=t.0                   |
| Sub-workflows for cold email and sales letter writing improve modularity and code reuse.         | Workflow IDs referenced: Cold Email Copywriter (ZtScEU9B5PNPnIcP), Sales Letter Agent (HQuj3iBV1PlI0HY1)         |
| The system prompt in Copy Assistant defines detailed SOPs and examples to improve intent detection accuracy. | Critical for ensuring correct routing and high-quality output.                                                |

---

**Disclaimer:**  
The text provided is derived exclusively from an automated workflow created with n8n, a tool for integration and automation. This processing strictly complies with applicable content policies and contains no illegal, offensive, or protected material. All data handled is legal and public.

---