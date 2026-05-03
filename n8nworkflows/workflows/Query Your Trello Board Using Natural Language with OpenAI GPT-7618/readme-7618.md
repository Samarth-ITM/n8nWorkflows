Query Your Trello Board Using Natural Language with OpenAI GPT

https://n8nworkflows.xyz/workflows/query-your-trello-board-using-natural-language-with-openai-gpt-7618


# Query Your Trello Board Using Natural Language with OpenAI GPT

### 1. Workflow Overview

This workflow enables natural language querying of a Trello board using OpenAI’s GPT language model. It serves as a conversational assistant that fetches live data from a Trello board (boards, lists, cards), aggregates this context, and uses AI to interpret user chat inputs to provide concise answers or summaries about the board. The workflow is ideal for project standups, status checks, and task management without leaving the chat interface.

Logical blocks:

- **1.1 Input Reception**: Captures the user’s chat message as input.
- **1.2 Trello Data Retrieval**: Fetches Trello board details, lists, and cards.
- **1.3 Data Transformation & Aggregation**: Maps relevant fields and aggregates all Trello data into a single dataset.
- **1.4 Chat Context Merging**: Combines live Trello data with chat input.
- **1.5 AI Processing & Memory**: Uses OpenAI GPT-enabled agent with memory to analyze the combined data and user query, producing a response.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview**: This block listens for incoming chat messages that trigger the workflow, initiating the processing chain.
- **Nodes Involved**:  
  - When chat message received
- **Node Details**:  
  - **When chat message received**  
    - Type: LangChain Chat Trigger node  
    - Role: Entry point; listens on a webhook for chat input messages  
    - Configuration: Default options, webhook ID assigned internally  
    - Inputs: External chat message (webhook trigger)  
    - Outputs: Emits JSON containing chat input data  
    - Edge cases: Incoming message format errors, webhook connectivity issues  
    - No sub-workflow references  

#### 2.2 Trello Data Retrieval

- **Overview**: Retrieves Trello board metadata, all lists on the board, and all cards within those lists based on the provided board URL.
- **Nodes Involved**:  
  - Get Board3  
  - Get Lists3  
  - Get Cards3
- **Node Details**:  
  - **Get Board3**  
    - Type: Trello API node (resource: board, operation: get)  
    - Role: Resolves Trello board from a URL to fetch board ID and metadata  
    - Configuration: Receives board URL in URL mode; uses Trello API credentials  
    - Inputs: Trigger from chat message reception block  
    - Outputs: Board JSON including ID and name  
    - Edge cases: Invalid URL, API credential errors, rate limiting  
  - **Get Lists3**  
    - Type: Trello API node (resource: list, operation: getAll)  
    - Role: Retrieves all lists on the board by board ID  
    - Configuration: Uses board ID from Get Board3 output; same Trello credentials  
    - Inputs: Board ID JSON from Get Board3  
    - Outputs: Array of lists JSON objects  
    - Edge cases: Board ID missing or invalid, API timeout  
  - **Get Cards3**  
    - Type: Trello API node (resource: list, operation: getCards)  
    - Role: For each list, retrieves all cards  
    - Configuration: Uses list ID from Get Lists3 output; same credentials  
    - Inputs: List IDs JSON from Get Lists3  
    - Outputs: Cards JSON per list  
    - Edge cases: Large boards causing timeouts, missing permissions  

#### 2.3 Data Transformation & Aggregation

- **Overview**: Transforms Trello data fields into a structured format suitable for AI context and aggregates all card data from multiple lists into one dataset.
- **Nodes Involved**:  
  - Map Fields3  
  - Combine into One1
- **Node Details**:  
  - **Map Fields3**  
    - Type: Set node  
    - Role: Maps raw Trello card and list data into named fields (Board Name, List Name, Task Name, Task Description, Due badge)  
    - Configuration: Uses expressions to extract data from previous nodes:  
      - Board Name: from Get Board3  
      - List Name: from Get Lists3  
      - Task Name, Description, Due (badges.due): from current card JSON  
    - Inputs: Cards JSON from Get Cards3  
    - Outputs: Structured JSON with mapped fields  
    - Edge cases: Missing fields in Trello cards/lists, expression syntax errors  
  - **Combine into One1**  
    - Type: Aggregate node  
    - Role: Aggregates all mapped card data into a single combined array for AI processing  
    - Configuration: Aggregates all incoming items without filtering  
    - Inputs: Multiple mapped card JSON items from Map Fields3  
    - Outputs: Single aggregated JSON array  
    - Edge cases: Extremely large boards may cause performance issues  

#### 2.4 Chat Context Merging

- **Overview**: Merges the aggregated Trello data with the incoming chat message for the AI to analyze both data and user query simultaneously.
- **Nodes Involved**:  
  - Merge With Chat  
  - Edit Fields1
- **Node Details**:  
  - **Merge With Chat**  
    - Type: Merge node  
    - Role: Combines the aggregated Trello data and chat input into a single data stream  
    - Configuration: Mode set to ‘combine’, combining all incoming data streams  
    - Inputs: Aggregated Trello data from Combine into One1 and chat input from When chat message received  
    - Outputs: Combined JSON object containing chat and Trello data  
    - Edge cases: Mismatched data keys or missing data  
  - **Edit Fields1**  
    - Type: Set node  
    - Role: Sets a string field named ‘trello’ with the aggregated Trello data extracted from merged JSON  
    - Configuration: Expression assigns `{{ $json.data }}` to new field `trello`  
    - Inputs: Combined JSON from Merge With Chat  
    - Outputs: JSON with explicit ‘trello’ field for AI node consumption  
    - Edge cases: Data extraction errors if input format changes  

#### 2.5 AI Processing & Memory

- **Overview**: Uses a LangChain agent with OpenAI GPT to analyze the merged Trello data and chat input, incorporating session-based memory to maintain conversational context.
- **Nodes Involved**:  
  - Simple Memory  
  - Trello Chatbot  
  - OpenAI Chat Model1
- **Node Details**:  
  - **Simple Memory**  
    - Type: LangChain Memory Buffer Window node  
    - Role: Maintains conversation state per session, keyed by sessionId from merged chat data  
    - Configuration: Uses `sessionKey` expression `={{ $('Merge With Chat').item.json.sessionId }}` to identify sessions  
    - Inputs: AI memory input stream from Trello Chatbot outputs  
    - Outputs: Provides conversation memory context to Trello Chatbot  
    - Edge cases: Missing sessionId fields, memory size limits  
  - **Trello Chatbot**  
    - Type: LangChain Agent node  
    - Role: Combines Trello data and chat input text into a prompt for AI, with an optional system message to summarize the board  
    - Configuration:  
      - Text prompt: `trello board aggregation: {{ $json.trello }} Chat message: {{ $('Merge With Chat').item.json.chatInput }}`  
      - System message: “Summarize this board”  
      - Prompt type: define  
    - Inputs: Edited fields with Trello data and chat input, plus AI language model and memory inputs  
    - Outputs: AI-generated response text  
    - Edge cases: API limits, malformed prompts, memory synchronization issues  
  - **OpenAI Chat Model1**  
    - Type: LangChain OpenAI Chat Model node  
    - Role: Executes the GPT-5-Nano model to generate AI outputs for Trello Chatbot  
    - Configuration: Model set to “gpt-5-nano,” uses OpenAI API credentials configured in n8n  
    - Inputs: Prompt from Trello Chatbot agent  
    - Outputs: AI text completions for chatbot responses  
    - Edge cases: API key errors, billing issues, model availability  

---

### 3. Summary Table

| Node Name                 | Node Type                            | Functional Role                      | Input Node(s)              | Output Node(s)           | Sticky Note                                                                                                                       |
|---------------------------|------------------------------------|------------------------------------|----------------------------|--------------------------|----------------------------------------------------------------------------------------------------------------------------------|
| When chat message received | LangChain Chat Trigger              | Entry point; receives chat input   | -                          | Get Board3, Merge With Chat |                                                                                                                                  |
| Get Board3                | Trello API                         | Fetch Trello board by URL           | When chat message received | Get Lists3               | See Sticky Note4 & Sticky Note55 for Trello API setup instructions                                                               |
| Get Lists3                | Trello API                         | Get all lists in board              | Get Board3                 | Get Cards3               |                                                                                                                                  |
| Get Cards3                | Trello API                         | Get cards for each list             | Get Lists3                 | Map Fields3              |                                                                                                                                  |
| Map Fields3               | Set                               | Map and rename Trello card fields  | Get Cards3                 | Combine into One1         |                                                                                                                                  |
| Combine into One1          | Aggregate                         | Aggregate all card data             | Map Fields3                | Merge With Chat           |                                                                                                                                  |
| Merge With Chat            | Merge                            | Combine chat and Trello data       | When chat message received, Combine into One1 | Edit Fields1           |                                                                                                                                  |
| Edit Fields1               | Set                               | Prepare Trello data field for AI   | Merge With Chat            | Trello Chatbot            |                                                                                                                                  |
| Simple Memory              | LangChain Memory Buffer Window    | Maintain chat session memory       | Trello Chatbot (ai_memory) | Trello Chatbot (memory)   |                                                                                                                                  |
| Trello Chatbot             | LangChain Agent                   | Process prompt with AI & memory    | Edit Fields1, Simple Memory, OpenAI Chat Model1 | None (terminal)          |                                                                                                                                  |
| OpenAI Chat Model1         | LangChain OpenAI Chat Model       | Generate AI responses (GPT-5-Nano) | Trello Chatbot (ai_languageModel) | Trello Chatbot (languageModel) | See Sticky Note54 for OpenAI API setup instructions                                                                               |
| Sticky Note4               | Sticky Note                      | Setup instructions for Trello & OpenAI | -                          | -                        | Contains detailed Trello & OpenAI connection setup instructions and contact information                                           |
| Sticky Note53              | Sticky Note                      | Overview description of workflow   | -                          | -                        | General introduction to the workflow’s purpose and capabilities                                                                  |
| Sticky Note54              | Sticky Note                      | OpenAI API connection instructions | -                          | -                        | See OpenAI setup steps                                                                                                            |
| Sticky Note55              | Sticky Note                      | Trello API credential setup steps  | -                          | -                        | See Trello API key and token setup                                                                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Trigger Node**  
   - Add a **LangChain Chat Trigger** node named `When chat message received`.  
   - Configure it with default options; this node listens for incoming chat messages via webhook.

2. **Set Up Trello API Credentials**  
   - In n8n Credentials, create a new **Trello API** credential.  
   - Enter your Trello **API Key** and **Token** (from https://trello.com/app-key).  

3. **Add Trello Board Node (`Get Board3`)**  
   - Add a **Trello** node, set resource to `Board`, operation to `Get`.  
   - Set the **ID** parameter mode to `URL` and paste your Trello board URL (e.g., `https://trello.com/b/DCpuJbnd/administrative-tasks`).  
   - Attach the Trello credentials.  
   - Connect `When chat message received` main output to this node’s input.

4. **Add Trello Lists Node (`Get Lists3`)**  
   - Add a **Trello** node, resource `List`, operation `Get All`.  
   - Set **ID** parameter to `={{ $json.id }}` (board ID from previous node).  
   - Use same Trello credentials.  
   - Connect output of `Get Board3` to input of this node.

5. **Add Trello Cards Node (`Get Cards3`)**  
   - Add a **Trello** node, resource `List`, operation `Get Cards`.  
   - Set **ID** parameter to `={{ $json.id }}` (list ID from previous node).  
   - Use same credentials.  
   - Connect output of `Get Lists3` to input of this node.

6. **Add Data Mapping Node (`Map Fields3`)**  
   - Add a **Set** node.  
   - Configure assignments:  
     - Board Name: `={{ $('Get Board3').item.json.name }}`  
     - List Name: `={{ $('Get Lists3').item.json.name }}`  
     - Task Name: `={{ $json.name }}`  
     - Task Description: `={{ $json.desc }}`  
     - badges.due: `={{ $json.badges.due }}`  
   - Connect output of `Get Cards3` to this node.

7. **Add Data Aggregation Node (`Combine into One1`)**  
   - Add an **Aggregate** node.  
   - Set aggregate method to `Aggregate All Item Data`.  
   - Connect output of `Map Fields3` to this node.

8. **Add Merge Node (`Merge With Chat`)**  
   - Add a **Merge** node, mode set to `Combine`.  
   - Connect output of `When chat message received` to input 1.  
   - Connect output of `Combine into One1` to input 2.

9. **Add Data Preparation Node (`Edit Fields1`)**  
   - Add a **Set** node.  
   - Create a string field `trello` with value: `={{ $json.data }}` (the aggregated Trello data).  
   - Connect output of `Merge With Chat` to this node.

10. **Set Up OpenAI API Credentials**  
    - Create an **OpenAI** credential in n8n with your API key (from https://platform.openai.com/api-keys).  
    - Ensure billing is set up and funds are available.

11. **Add OpenAI Chat Model Node (`OpenAI Chat Model1`)**  
    - Add a **LangChain OpenAI Chat Model** node.  
    - Select model `gpt-5-nano`.  
    - Attach OpenAI credentials.  
    - This node will be linked as AI language model input to the agent node.

12. **Add AI Memory Node (`Simple Memory`)**  
    - Add a **LangChain Memory Buffer Window** node.  
    - Set `sessionKey` to: `={{ $('Merge With Chat').item.json.sessionId }}` to track conversation sessions.  
    - Connect this node to provide AI memory input to the agent node.

13. **Add Agent Node (`Trello Chatbot`)**  
    - Add a **LangChain Agent** node.  
    - Set prompt text to:  
      `trello board aggregation: {{ $json.trello }} Chat message: {{ $('Merge With Chat').item.json.chatInput }}`  
    - Set system message to: “Summarize this board”  
    - Set prompt type to `define`.  
    - Connect inputs:  
      - Main input from `Edit Fields1`  
      - AI language model from `OpenAI Chat Model1`  
      - AI memory from `Simple Memory`  
    - This node produces the final AI-generated response.

14. **Connect Workflow Outputs**  
    - The output from `Trello Chatbot` is terminal and can be sent back as chat response to users.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                         | Context or Link                                                                                                             |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------|
| Detailed setup instructions for Trello API key and token generation.                                                                                               | Sticky Note4 and Sticky Note55 provide step-by-step Trello API credential setup instructions.                                |
| Instructions for setting up OpenAI API credentials, including billing and key creation.                                                                             | Sticky Note4 and Sticky Note54 cover OpenAI API setup with links to https://platform.openai.com/api-keys and billing page. |
| Workflow enables conversational interaction with Trello boards without opening Trello, useful for standups and quick status queries.                              | Sticky Note53 provides a high-level description of workflow purpose and use cases.                                           |
| Contact details for workflow author Robert Breen, including email and LinkedIn profile.                                                                             | Included in Sticky Note4.                                                                                                   |
| Uses n8n’s LangChain integration nodes for chat triggers, memory management, and AI language modeling, leveraging GPT-5-Nano.                                       | Requires n8n version supporting LangChain nodes with version >=1.2 for OpenAI Chat Model and version 1.3+ for chat trigger. |

---

**Disclaimer:**  
The provided text and workflow stem exclusively from an n8n automated process. It complies with all content policies and contains no illegal or protected material. All data processed is public and legal.