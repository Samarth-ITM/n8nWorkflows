ERP AI chatbot for Odoo sales module with OpenAI

https://n8nworkflows.xyz/workflows/erp-ai-chatbot-for-odoo-sales-module-with-openai-2325


# ERP AI chatbot for Odoo sales module with OpenAI

### 1. Workflow Overview

This workflow implements an AI-powered chatbot interface for the Odoo sales module, enabling users to query and receive conversational summaries and insights about their sales opportunities without manual data extraction or complex queries. It leverages OpenAI language models to process sales data from Odoo, generate concise summaries, and maintain an updated contextual knowledge base that the chatbot uses for conversations.

The workflow is composed of three primary logical blocks:

- **1.1 Data Retrieval and Summarization:** Periodically fetches all sales opportunities from Odoo, aggregates them, and generates a summarized text using OpenAI's GPT-4 Turbo model. The summary is cached in a file to serve as a knowledge base for chatbot conversations.

- **1.2 Chatbot Interaction and Contextual Memory:** Handles incoming chat requests, loads the latest summary as context, maintains session-based memory buffers for conversational continuity, and processes user queries with OpenAI chat models augmented by tools like a calculator.

- **1.3 Scheduling and Maintenance:** Uses a schedule trigger to refresh the sales data summary regularly, ensuring chatbot responses are based on up-to-date information.

---

### 2. Block-by-Block Analysis

---

#### 2.1 Data Retrieval and Summarization

**Overview:**  
This block periodically retrieves all sales opportunities from the connected Odoo instance, merges the data, and uses OpenAI's GPT-4 Turbo model to generate a concise, structured summary. The summary is saved to a local file ("cache.txt") for subsequent use by the chatbot.

**Nodes Involved:**  
- Schedule Trigger  
- Get All Opportunities from Odoo  
- Merge Opportunities  
- OpenAI Summarization Model  
- Summarize Opportunities  
- Convert to File  
- Save Summary to File  
- Read Summary From File (also used in chatbot block for sharing)

**Node Details:**

- **Schedule Trigger**  
  - *Type & Role:* n8n built-in node to trigger the workflow on a repeating schedule.  
  - *Configuration:* Default schedule with an interval array (likely every minute or hour; exact interval not specified).  
  - *Input/Output:* No input; triggers "Get All Opportunities from Odoo".  
  - *Edge Cases:* Scheduling failures or missed triggers; ensure n8n is running continuously.

- **Get All Opportunities from Odoo**  
  - *Type & Role:* Odoo API node to fetch all sales opportunities.  
  - *Configuration:* Retrieves fields: won_status, description, email_from, contact_name, expected_revenue; returns all records.  
  - *Credentials:* Requires configured Odoo API credentials.  
  - *Input/Output:* Triggered by schedule; outputs list of opportunities.  
  - *Edge Cases:* API auth errors, connectivity issues, large data sets causing timeouts.

- **Merge Opportunities**  
  - *Type & Role:* Aggregation node that consolidates all opportunity items into a single data array.  
  - *Configuration:* Aggregates all item data into one object/array.  
  - *Input/Output:* Input from Odoo node; output to summarization chain.  
  - *Edge Cases:* Empty datasets or malformed data.

- **OpenAI Summarization Model**  
  - *Type & Role:* Language model node using GPT-4 Turbo for text generation.  
  - *Configuration:* Model set to "gpt-4-turbo" with default options.  
  - *Credentials:* Requires OpenAI API credentials.  
  - *Input/Output:* Receives merged opportunity data for summarization; outputs text summary.  
  - *Edge Cases:* API rate limits, authentication errors, network timeouts.

- **Summarize Opportunities**  
  - *Type & Role:* LangChain summarization chain node orchestrating the prompt and summarization process.  
  - *Configuration:* Uses a custom prompt instructing the model to write a summary including won status, expected revenue, and descriptions for each opportunity separately.  
  - *Input/Output:* Input from merged data; output summary text to be converted to file.  
  - *Edge Cases:* Prompt failures if data is invalid or too large.

- **Convert to File**  
  - *Type & Role:* Converts the summary text to a text file format.  
  - *Configuration:* Converts "response.text" property to a text file.  
  - *Input/Output:* Input is summary text; output is file data.  
  - *Edge Cases:* Conversion errors if input is missing.

- **Save Summary to File**  
  - *Type & Role:* Writes the summary text file to disk with filename "cache.txt".  
  - *Configuration:* Overwrites existing file, no append.  
  - *Input/Output:* Input is file data from conversion; output triggers reading summary (loop).  
  - *Edge Cases:* File write permissions issues, disk space.

- **Read Summary From File**  
  - *Type & Role:* Reads the cached summary file to feed the chatbot context.  
  - *Configuration:* Reads "cache.txt" file without special options.  
  - *Input/Output:* Reads file; outputs file metadata and content for conditional processing.  
  - *Edge Cases:* File not found (on first run), file read errors.

---

#### 2.2 Chatbot Interaction and Contextual Memory

**Overview:**  
This block handles incoming chat messages from users via a public chat trigger webhook, loads the latest sales summary as context, maintains session-specific conversational memory buffers, and processes user questions using OpenAI chat models and auxiliary tools.

**Nodes Involved:**  
- Chat Trigger  
- Read Summary From File (shared with summarization block)  
- If Summary Exists  
- Extract Text From File  
- AI Conversational Agent  
- OpenAI Chat Model  
- Window Buffer Memory  
- Calculator  

**Node Details:**

- **Chat Trigger**  
  - *Type & Role:* LangChain chat webhook node that receives chat input from users.  
  - *Configuration:* Publicly accessible webhook enabled (toggle available).  
  - *Input/Output:* Receives chat input and session ID; outputs to read summary node.  
  - *Edge Cases:* Unauthorized access if public; webhook downtime.

- **Read Summary From File**  
  - *Role:* Reads the cached summary file to provide context for chat.  
  - *See section 2.1 for details.*

- **If Summary Exists**  
  - *Type & Role:* Conditional node to check if the summary file exists and is non-empty.  
  - *Configuration:* Checks if the "fileName" property exists in the read file data.  
  - *Input/Output:* Routes flow either to extract text or to refresh data from Odoo.  
  - *Edge Cases:* False negatives if file metadata is inconsistent.

- **Extract Text From File**  
  - *Type & Role:* Extracts the text content from the summary file data for use as context.  
  - *Configuration:* Operation set to extract text from file input.  
  - *Input/Output:* Input from file read node; output to AI Conversational Agent.  
  - *Edge Cases:* Extraction errors if file content is corrupted.

- **AI Conversational Agent**  
  - *Type & Role:* LangChain agent node that processes the user's chat input using conversational AI, tools, and memory.  
  - *Configuration:*  
    - Uses the user's chat input as text.  
    - Configured with a "conversationalAgent" type.  
    - Provides a custom prompt instructing the agent to use provided tools and context (the summary data) to answer questions, formatted as a JSON code snippet.  
    - Human message includes instructions about tools usage and context limitations.  
  - *Input/Output:* Receives user input and context; outputs agent's conversational JSON response.  
  - *Edge Cases:* Prompt injection risks, response formatting errors.

- **OpenAI Chat Model**  
  - *Type & Role:* OpenAI chat model node (e.g., GPT-4 Turbo chat) used as the language model backend for the conversational agent.  
  - *Configuration:* Default options with OpenAI API credentials.  
  - *Input/Output:* Connected as the language model for the AI Conversational Agent node.  
  - *Edge Cases:* API limits, auth failures.

- **Window Buffer Memory**  
  - *Type & Role:* LangChain memory buffer node that maintains a sliding window of conversation history per session key.  
  - *Configuration:* Session key derived from the chat trigger node's session ID; uses custom key session ID type.  
  - *Input/Output:* Connected as the memory input for the AI Conversational Agent node to maintain conversational context.  
  - *Edge Cases:* Memory size limits, session ID consistency.

- **Calculator**  
  - *Type & Role:* LangChain tool node providing calculator capabilities to the conversational agent for numeric computations.  
  - *Configuration:* Default (no parameters).  
  - *Input/Output:* Connected as a tool for the AI Conversational Agent node.  
  - *Edge Cases:* Calculator errors if invalid input expressions given.

---

#### 2.3 Scheduling and Maintenance

**Overview:**  
This block ensures the sales opportunity summary is kept up-to-date by periodically triggering data retrieval and summarization operations.

**Nodes Involved:**  
- Schedule Trigger  
- Get All Opportunities from Odoo  
- Merge Opportunities  
- OpenAI Summarization Model  
- Summarize Opportunities  
- Convert to File  
- Save Summary to File  
- Read Summary From File (used downstream)

**Note:** This block overlaps significantly with Block 2.1 but is explicitly responsible for refreshing the summary on a schedule.

---

### 3. Summary Table

| Node Name               | Node Type                                   | Functional Role                           | Input Node(s)                 | Output Node(s)               | Sticky Note                                                                                                               |
|-------------------------|---------------------------------------------|-----------------------------------------|------------------------------|------------------------------|---------------------------------------------------------------------------------------------------------------------------|
| Window Buffer Memory     | @n8n/n8n-nodes-langchain.memoryBufferWindow| Maintains session-based conversation memory | AI Conversational Agent (input memory) | AI Conversational Agent (memory) |                                                                                                                           |
| OpenAI Chat Model       | @n8n/n8n-nodes-langchain.lmChatOpenAi       | Provides chat language model for agent  | AI Conversational Agent (language model) | AI Conversational Agent (language model) |                                                                                                                           |
| Calculator              | @n8n/n8n-nodes-langchain.toolCalculator     | Calculator tool for conversational agent | AI Conversational Agent (tool) | AI Conversational Agent (tool) |                                                                                                                           |
| Schedule Trigger        | n8n-nodes-base.scheduleTrigger               | Triggers periodic refresh of sales data | None                         | Get All Opportunities from Odoo |                                                                                                                           |
| Convert to File         | n8n-nodes-base.convertToFile                  | Converts summary text to file format     | Summarize Opportunities       | Save Summary to File          |                                                                                                                           |
| Save Summary to File    | n8n-nodes-base.readWriteFile                  | Writes summary file cache                | Convert to File               | Read Summary From File        |                                                                                                                           |
| Get All Opportunities from Odoo | n8n-nodes-base.odoo                      | Fetches all sales opportunities from Odoo | Schedule Trigger             | Merge Opportunities           |                                                                                                                           |
| Read Summary From File  | n8n-nodes-base.readWriteFile                  | Reads cached summary for chatbot context | Save Summary to File, Chat Trigger | If Summary Exists             |                                                                                                                           |
| If Summary Exists       | n8n-nodes-base.if                             | Checks if summary file exists            | Read Summary From File        | Extract Text From File, Get All Opportunities from Odoo |                                                                                                                           |
| Merge Opportunities     | n8n-nodes-base.aggregate                       | Aggregates all opportunities into one   | Get All Opportunities from Odoo | Summarize Opportunities       |                                                                                                                           |
| Extract Text From File  | n8n-nodes-base.extractFromFile                 | Extracts plain text from summary file   | If Summary Exists             | AI Conversational Agent       |                                                                                                                           |
| AI Conversational Agent | @n8n/n8n-nodes-langchain.agent                 | Processes user chat input with AI       | Extract Text From File, Window Buffer Memory, Calculator, OpenAI Chat Model | Calculator, OpenAI Chat Model, Window Buffer Memory |                                                                                                                           |
| Summarize Opportunities | @n8n/n8n-nodes-langchain.chainSummarization   | Generates concise summary of sales data | Merge Opportunities           | Convert to File               |                                                                                                                           |
| OpenAI Summarization Model | @n8n/n8n-nodes-langchain.lmOpenAi           | Language model for summarization         | Summarize Opportunities       | Summarize Opportunities       |                                                                                                                           |
| Sticky Note             | n8n-nodes-base.stickyNote                      | Provides setup instructions and overview | None                         | None                         | # ERP chatbot for Odoo sales module  Set up steps:  * Configure the Odoo credentials  * Configure OpenAI credentials  * Toggle "Make Chat Publicly Available" from the Chat Trigger node. |
| Chat Trigger            | @n8n/n8n-nodes-langchain.chatTrigger           | Receives user chat input via webhook     | None                         | Read Summary From File        |                                                                                                                           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create "Schedule Trigger" node**  
   - Type: scheduleTrigger  
   - Configure a recurring interval (e.g., every 1 hour) to refresh sales data.

2. **Create "Get All Opportunities from Odoo" node**  
   - Type: odoo  
   - Credentials: Link to Odoo account credentials.  
   - Resource: opportunity  
   - Operation: getAll  
   - Fields: won_status, description, email_from, contact_name, expected_revenue  
   - Return all: true  
   - Connect output of "Schedule Trigger" to this node's input.

3. **Create "Merge Opportunities" node**  
   - Type: aggregate  
   - Operation: aggregate all item data into a single array/object.  
   - Connect output of "Get All Opportunities from Odoo" to this node.

4. **Create "OpenAI Summarization Model" node**  
   - Type: lmOpenAi  
   - Credentials: Link OpenAI API credentials.  
   - Model: gpt-4-turbo  
   - Connect output of "Merge Opportunities" to this node.

5. **Create "Summarize Opportunities" node**  
   - Type: chainSummarization  
   - Parameters: Use a custom prompt to instruct summarization:  
     ```
     Write a summary of the following:

     {{ JSON.stringify($json.data) }}

     Include important information such as won status and expected revenue for each opportunity. Also include a short description of each opportunity and keep opportunities separate.

     CONCISE SUMMARY:
     ```  
   - Connect output of "OpenAI Summarization Model" to this node.

6. **Create "Convert to File" node**  
   - Type: convertToFile  
   - Operation: toText  
   - Source property: response.text (summary text)  
   - Connect output of "Summarize Opportunities" to this node.

7. **Create "Save Summary to File" node**  
   - Type: readWriteFile  
   - Operation: write  
   - File name: cache.txt  
   - Append: false (overwrite)  
   - Connect output of "Convert to File" to this node.

8. **Create "Read Summary From File" node**  
   - Type: readWriteFile  
   - Operation: read  
   - File selector: cache.txt  
   - Connect output of "Save Summary to File" to this node (to enable re-use)  
   - Also to be connected later from chat trigger (see below).

9. **Create "If Summary Exists" node**  
   - Type: if  
   - Condition: Check if "fileName" exists in the read file output (string exists).  
   - Connect output of "Read Summary From File" to this node.

10. **Create "Extract Text From File" node**  
    - Type: extractFromFile  
    - Operation: text  
    - Connect "true" output of "If Summary Exists" to this node.

11. **Create "AI Conversational Agent" node**  
    - Type: langchain agent  
    - Text parameter: `={{ $('Chat Trigger').item.json.chatInput }}`  
    - Agent: conversationalAgent  
    - Prompt: Custom prompt that instructs the agent to use tools and to answer using only the context from the summary data, formatted as JSON code snippet (copy prompt from original node).  
    - Connect output of "Extract Text From File" to this node as context.  
    - Connect memory and tools (see below).

12. **Create "Chat Trigger" node**  
    - Type: langchain chatTrigger  
    - Enable "Make Chat Publicly Available" toggle.  
    - Webhook ID will be generated automatically.  
    - Connect output of "Chat Trigger" to "Read Summary From File" (entry point for chat requests).

13. **Connect "If Summary Exists" node outputs**  
    - True branch: to "Extract Text From File" (loads summary for context).  
    - False branch: to "Get All Opportunities from Odoo" (forces refresh if no summary).

14. **Create "Window Buffer Memory" node**  
    - Type: langchain memoryBufferWindow  
    - Session key: `={{ $('Chat Trigger').item.json.sessionId }}`  
    - Session ID type: customKey  
    - Connect output of "Window Buffer Memory" to "AI Conversational Agent" memory input.

15. **Create "OpenAI Chat Model" node**  
    - Type: langchain lmChatOpenAi  
    - Credentials: OpenAI API credentials.  
    - Connect output of "OpenAI Chat Model" to "AI Conversational Agent" language model input.

16. **Create "Calculator" node**  
    - Type: langchain toolCalculator  
    - Default configuration.  
    - Connect output of "Calculator" node to "AI Conversational Agent" ai_tool input.

17. **Connect "AI Conversational Agent" outputs**  
    - Connect "ai_tool", "ai_languageModel", and "ai_memory" inputs accordingly as shown above.

18. **Sticky Note Setup (Optional)**  
    - Add a sticky note with setup instructions:  
      ```
      # ERP chatbot for Odoo sales module

      Set up steps:
      * Configure the Odoo credentials
      * Configure OpenAI credentials
      * Toggle "Make Chat Publicly Available" from the Chat Trigger node.
      ```

---

### 5. General Notes & Resources

| Note Content                                                                                                  | Context or Link                                                                                                    |
|---------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------|
| The workflow requires configured credentials for both Odoo API and OpenAI API to operate correctly.           | Credential setup is mandatory before executing the workflow.                                                     |
| Make sure to toggle "Make Chat Publicly Available" in the Chat Trigger node to enable external chat access.   | This setting controls webhook accessibility; for private use, disable the toggle.                                 |
| The workflow uses LangChain nodes from n8n's langchain integration for agent, memory, and tool management.    | Refer to n8n LangChain node documentation for detailed usage and version compatibility.                           |
| The chatbot's JSON code snippet response format helps integration with UI clients expecting structured output. | Ensures predictable parsing and further processing of AI responses.                                              |
| OpenAI GPT-4 Turbo is used for both summarization and chat models for advanced language understanding.        | Requires OpenAI API key with GPT-4 access, which may have usage costs and rate limits.                            |
| The calculator tool enables the agent to perform numeric computations on user queries during chat sessions.   | Useful for queries involving sales figures, revenue calculations, or projections.                                |

---

This documentation provides a thorough understanding and stepwise reproduction guide of the "ERP AI chatbot for Odoo sales module" workflow, facilitating both manual modifications and automated analysis or deployment.