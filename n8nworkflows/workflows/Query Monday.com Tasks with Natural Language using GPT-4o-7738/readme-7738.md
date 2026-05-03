Query Monday.com Tasks with Natural Language using GPT-4o

https://n8nworkflows.xyz/workflows/query-monday-com-tasks-with-natural-language-using-gpt-4o-7738


# Query Monday.com Tasks with Natural Language using GPT-4o

### 1. Workflow Overview

This workflow enables natural language querying of Monday.com tasks using OpenAI’s GPT-4o model, providing users with an AI-powered assistant that fetches live task data from a specified Monday.com board and answers questions based solely on that data. It targets users who want to interact conversationally with their Monday.com tasks, such as asking about overdue items, task summaries, or status updates, without manually navigating the board.

The workflow is logically divided into the following functional blocks:

- **1.1 Input Reception:** Captures user queries via a chatbot trigger.
- **1.2 Data Retrieval:** Fetches tasks from a specified Monday.com board and group.
- **1.3 Data Processing and Formatting:** Extracts relevant fields from tasks, formats and aggregates them into a text block.
- **1.4 Context Management:** Maintains conversational context using memory buffers.
- **1.5 AI Processing:** Uses GPT-4o to interpret the user query in the context of the fetched task data and generates a relevant natural language response.
- **1.6 Output Delivery:** Sends the AI-generated answer back as the chatbot response.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
This block starts the workflow by receiving natural language questions from users through a chat trigger node designed for conversational AI workflows.

**Nodes Involved:**  
- Sample Chatbot

**Node Details:**  

- **Sample Chatbot**  
  - *Type:* LangChain Chat Trigger  
  - *Role:* Entry point that listens for incoming chat messages (webhook-based).  
  - *Configuration:* No special parameters beyond default; uses a webhook ID for external integration.  
  - *Input:* External chat message via webhook.  
  - *Output:* User’s chat input passed downstream.  
  - *Edge Cases:* Webhook not reachable, invalid payload, or missing chat input can cause failures.  
  - *Sub-workflow:* None.

---

#### 2.2 Data Retrieval

**Overview:**  
Fetches all items from a specific Monday.com board and group to provide the AI assistant with the most current task data.

**Nodes Involved:**  
- Get many items

**Node Details:**  

- **Get many items**  
  - *Type:* Monday.com API node  
  - *Role:* Retrieves all board items from a specified board and group.  
  - *Configuration:*  
    - Board ID: `9865886546`  
    - Group ID: `new_group29179`  
    - Operation: `getAll` on `boardItem` resource  
  - *Input:* Triggered after receiving chat input.  
  - *Output:* Raw Monday.com item data.  
  - *Credentials:* Monday.com API token required.  
  - *Edge Cases:* API authentication failure, rate limiting, empty group or board, network errors.  
  - *Sub-workflow:* None.

---

#### 2.3 Data Processing and Formatting

**Overview:**  
Extracts key task fields (name, status, due date), formats them, aggregates all items into a single data structure, and converts this to plain text for AI consumption.

**Nodes Involved:**  
- Set Fields1  
- Combine to One Field  
- Merge with Chat  
- Convert to text

**Node Details:**  

- **Set Fields1**  
  - *Type:* Set node  
  - *Role:* Selects and renames key fields from Monday.com item JSON for clarity.  
  - *Configuration:*  
    - Sets `name` to item’s name  
    - `Status` to text of second column value  
    - `Due Date` to value of third column  
  - *Input:* Monday.com item data  
  - *Output:* Simplified JSON with selected fields  
  - *Edge Cases:* Missing columns or unexpected data structure causing undefined values.

- **Combine to One Field**  
  - *Type:* Aggregate node  
  - *Role:* Aggregates all processed items into a single data array under one field.  
  - *Input:* Processed fields from `Set Fields1`  
  - *Output:* Aggregated data collection  
  - *Edge Cases:* Empty input array results in empty aggregation.

- **Merge with Chat**  
  - *Type:* Merge node  
  - *Role:* Combines aggregated task data with original chat input for joint processing.  
  - *Configuration:* Mode set to `combine`, combining all inputs into one item.  
  - *Input:* Aggregated task data and chat input  
  - *Output:* Single item containing both chat input and task data  
  - *Edge Cases:* Mismatched input counts causing merge issues.

- **Convert to text**  
  - *Type:* Set node  
  - *Role:* Converts the aggregated task data into a string field named `data` for AI prompt.  
  - *Input:* Combined JSON data from merge node  
  - *Output:* JSON with `data` string field  
  - *Edge Cases:* Conversion may fail if data is undefined or malformatted.

---

#### 2.4 Context Management

**Overview:**  
Maintains conversational context across user interactions by storing and retrieving memory buffers keyed by session ID.

**Nodes Involved:**  
- Simple Memory3

**Node Details:**  

- **Simple Memory3**  
  - *Type:* LangChain Memory Buffer (Window)  
  - *Role:* Stores conversation history for context preservation in multi-turn conversations.  
  - *Configuration:*  
    - Session key is dynamically set based on session ID from merged chat item (`={{ $('Merge with Chat').item.json.sessionId }}`)  
  - *Input:* AI agent node output (chat completion)  
  - *Output:* Memory context for AI model  
  - *Edge Cases:* Missing or invalid session ID causes context loss, potentially degrading conversational quality.

---

#### 2.5 AI Processing

**Overview:**  
Generates AI responses by feeding the user's chat input and the formatted Monday.com task data into GPT-4o, leveraging conversational memory.

**Nodes Involved:**  
- OpenAI Chat Model7  
- Chat with Monday.com

**Node Details:**  

- **OpenAI Chat Model7**  
  - *Type:* LangChain OpenAI Chat Model node  
  - *Role:* Provides GPT-4o language model inference.  
  - *Configuration:*  
    - Model set to `gpt-4o`  
    - Uses OpenAI credentials configured in n8n  
  - *Input:* Context from memory and prompt from agent node  
  - *Output:* AI-generated message text  
  - *Edge Cases:* API key invalid, quota exceeded, network timeouts, rate limiting.

- **Chat with Monday.com**  
  - *Type:* LangChain Agent node  
  - *Role:* Combines chat input and task data into a prompt and instructs the AI on how to use the data.  
  - *Configuration:*  
    - Prompt text built as: `Chat Message: {{ $('Merge with Chat').item.json.chatInput }} Tasks:  {{ $json.data }}`  
    - System message instructs: "Use the tasks data to answer the chat message"  
  - *Input:* Combined chat input and task data from `Convert to text`  
  - *Output:* Prompt for OpenAI model  
  - *Edge Cases:* Missing data fields, malformed prompt, or unexpected input can cause poor AI responses.

---

#### 2.6 Output Delivery

**Overview:**  
Final AI-generated answer is produced as the output of the agent node, ready for delivery back to the chat interface.

**Nodes Involved:**  
- Chat with Monday.com (output stage)

**Node Details:**  

- **Chat with Monday.com** (output)  
  - *Type:* LangChain Agent node  
  - *Role:* Produces the final AI response.  
  - *Output:* Ends the workflow with the AI response to the user’s query.  
  - *Edge Cases:* Failure at prior nodes or empty AI output results in no response.

---

### 3. Summary Table

| Node Name           | Node Type                                  | Functional Role                   | Input Node(s)           | Output Node(s)       | Sticky Note                                                                                  |
|---------------------|--------------------------------------------|---------------------------------|------------------------|----------------------|----------------------------------------------------------------------------------------------|
| Sample Chatbot       | @n8n/n8n-nodes-langchain.chatTrigger       | Receives user chat input         | —                      | Get many items        |                                                                                              |
| Get many items       | n8n-nodes-base.mondayCom                    | Fetches Monday.com tasks         | Sample Chatbot          | Set Fields1           |                                                                                              |
| Set Fields1          | n8n-nodes-base.set                          | Extracts key task fields         | Get many items          | Combine to One Field  |                                                                                              |
| Combine to One Field | n8n-nodes-base.aggregate                     | Aggregates task data             | Set Fields1             | Merge with Chat       |                                                                                              |
| Merge with Chat      | n8n-nodes-base.merge                        | Combines chat input with data    | Combine to One Field, Sample Chatbot | Convert to text     |                                                                                              |
| Convert to text      | n8n-nodes-base.set                          | Converts aggregated data to text | Merge with Chat         | Chat with Monday.com  |                                                                                              |
| Chat with Monday.com | @n8n/n8n-nodes-langchain.agent             | Builds prompt & generates AI answer | Convert to text, OpenAI Chat Model7 | —                    |                                                                                              |
| OpenAI Chat Model7   | @n8n/n8n-nodes-langchain.lmChatOpenAi      | GPT-4o AI model inference       | Chat with Monday.com (ai_languageModel) | Chat with Monday.com |                                                                                              |
| Simple Memory3       | @n8n/n8n-nodes-langchain.memoryBufferWindow | Maintains conversation memory   | Chat with Monday.com (ai_memory) | OpenAI Chat Model7   |                                                                                              |
| Sticky Note65        | n8n-nodes-base.stickyNote                   | Documentation / User guidance   | —                      | —                    | # 💬 Chat with Monday.com (n8n + OpenAI)... The assistant fetches real data from your Monday.com board |
| Sticky Note23        | n8n-nodes-base.stickyNote                   | Setup instructions              | —                      | —                    | ## ⚙️ Setup Instructions... OpenAI and Monday.com connection details with links             |
| Sticky Note70        | n8n-nodes-base.stickyNote                   | Setup instructions (Monday.com) | —                      | —                    | ### 2️⃣ Connect Monday.com Node... Monday API Token and credential setup instructions        |
| Sticky Note25        | n8n-nodes-base.stickyNote                   | Setup instructions (OpenAI)     | —                      | —                    | ### 1️⃣ Set Up OpenAI Connection... Add funds and add API key to n8n credentials             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add the "Sample Chatbot" node:**  
   - Type: `@n8n/n8n-nodes-langchain.chatTrigger`  
   - Assign a webhook ID or leave default to receive chat messages.  
   - No special parameters required.

3. **Add the "Get many items" node:**  
   - Type: `n8n-nodes-base.mondayCom`  
   - Set credentials to your Monday.com API Token credential.  
   - Set parameters:  
     - Board ID: `9865886546` (replace with your board ID)  
     - Group ID: `new_group29179` (replace with your group ID)  
     - Resource: `boardItem`  
     - Operation: `getAll`  
   - Connect output of `Sample Chatbot` to input of this node.

4. **Add "Set Fields1" node:**  
   - Type: `n8n-nodes-base.set`  
   - Configure to assign:  
     - `name` = `{{$json["name"]}}`  
     - `Status` = `{{$json["column_values"][1]["text"]}}`  
     - `Due Date` = `{{$json["column_values"][2]["value"]}}`  
   - Connect output of `Get many items` to input of this node.

5. **Add "Combine to One Field" node:**  
   - Type: `n8n-nodes-base.aggregate`  
   - Set aggregate mode to combine all items into one array field (default `aggregateAllItemData`).  
   - Connect output of `Set Fields1` to input of this node.

6. **Add "Merge with Chat" node:**  
   - Type: `n8n-nodes-base.merge`  
   - Mode: `combine` (combine all inputs into one item)  
   - Connect output of `Combine to One Field` to one input, and output of `Sample Chatbot` to the other input.

7. **Add "Convert to text" node:**  
   - Type: `n8n-nodes-base.set`  
   - Add assignment:  
     - `data` = `={{ $json["data"] }}` (where `data` is the aggregated field from previous merge)  
   - Connect output of `Merge with Chat` to input of this node.

8. **Add "Chat with Monday.com" node:**  
   - Type: `@n8n/n8n-nodes-langchain.agent`  
   - Parameters:  
     - Prompt text:  
       `Chat Message: {{ $('Merge with Chat').item.json.chatInput }} Tasks: {{ $json.data }}`  
     - System Message: `"Use the tasks data to answer the chat message"`  
     - Prompt Type: `define`  
   - Connect output of `Convert to text` to input of this node.

9. **Add "Simple Memory3" node:**  
   - Type: `@n8n/n8n-nodes-langchain.memoryBufferWindow`  
   - Session Key: `={{ $('Merge with Chat').item.json.sessionId }}` (to uniquely identify chat sessions)  
   - Connect the AI agent output (`Chat with Monday.com`) to the memory input.

10. **Add "OpenAI Chat Model7" node:**  
    - Type: `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
    - Model: `gpt-4o`  
    - Credentials: Use your OpenAI API key credential.  
    - Connect memory output (`Simple Memory3`) to this node’s input (ai_languageModel).  
    - Connect this node’s output back to the AI agent node (`Chat with Monday.com`) input (ai_languageModel).

11. **Finalize connections:**  
    - Ensure the output of `Chat with Monday.com` node is the final output of the workflow.  
    - The workflow triggers on the webhook event from the `Sample Chatbot`.

12. **Credential Setup:**  
    - In n8n credentials, add your OpenAI API key for the OpenAI nodes.  
    - Add your Monday.com Personal API Token for the Monday.com node.  

13. **Test the workflow:**  
    - Send a natural language query through the webhook endpoint.  
    - Verify that tasks data is pulled and the AI generates relevant answers based on live Monday.com data.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                               | Context or Link                                                                                     |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| This workflow allows natural language queries on live Monday.com task data using GPT-4o, combining n8n automation with LangChain AI capabilities.                                                                                                                                         | Workflow purpose                                                                                   |
| Setup instructions include adding OpenAI API key and billing setup: https://platform.openai.com/api-keys and https://platform.openai.com/settings/organization/billing/overview.                                                                                                             | Sticky Note23 and Sticky Note25                                                                    |
| Monday.com API token generation and integration instructions: https://developer.monday.com/api-reference/docs/authentication                                                                                                                                                                | Sticky Note23 and Sticky Note70                                                                    |
| Contact for customization help: Robert Breen, robert@ynteractive.com, https://www.linkedin.com/in/robert-breen-29429625/, https://ynteractive.com                                                                                                                                          | Sticky Note23                                                                                      |
| The workflow depends on correct session ID propagation for memory context; ensure your chatbot interface passes session identifiers consistently to maintain conversation state.                                                                                                           | Context preservation note from Simple Memory3 block                                               |

---

**Disclaimer:**  
The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.