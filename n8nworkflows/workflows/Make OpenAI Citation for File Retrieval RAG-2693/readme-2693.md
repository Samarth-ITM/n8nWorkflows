Make OpenAI Citation for File Retrieval RAG

https://n8nworkflows.xyz/workflows/make-openai-citation-for-file-retrieval-rag-2693


# Make OpenAI Citation for File Retrieval RAG

### 1. Workflow Overview

This workflow, titled **Make OpenAI Citation for File Retrieval RAG**, is designed to enhance the output of an OpenAI assistant by integrating citation and source references from a vector store of files. It targets use cases where users want the assistant’s responses to include dynamically generated citations linked to original source files, formatted optionally in Markdown or HTML. This is particularly useful to avoid strange characters in the output and to provide clear, numbered references.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception and Trigger:** Captures user input via a chat trigger node integrated within n8n.
- **1.2 OpenAI Assistant Query with Vector Store:** Sends the input to an OpenAI assistant configured with a vector store for file retrieval.
- **1.3 Thread Content Retrieval:** Retrieves the full conversation thread content from OpenAI to obtain all citations.
- **1.4 Data Splitting and Extraction:** Splits the retrieved thread content into individual messages, then further splits messages into content and citations.
- **1.5 Citation File Metadata Retrieval:** For each citation, fetches the corresponding file metadata (filename) from OpenAI.
- **1.6 Data Regularization and Aggregation:** Normalizes the citation data and aggregates all citation entries into a single data structure.
- **1.7 Output Formatting:** Replaces citation texts in the assistant’s output with formatted references (Markdown by default), with an optional step to convert Markdown to HTML.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Trigger

- **Overview:**  
  This block initiates the workflow by capturing chat input within n8n, enabling interactive testing and use of the assistant.

- **Nodes Involved:**  
  - Create a simple Trigger to have the Chat button within N8N

- **Node Details:**  
  - **Create a simple Trigger to have the Chat button within N8N**  
    - Type: `@n8n/n8n-nodes-langchain.chatTrigger`  
    - Role: Provides a chat interface trigger inside n8n to start the workflow on user input.  
    - Configuration: Default options, webhook ID assigned for external access.  
    - Inputs: None (trigger node).  
    - Outputs: Connects to the OpenAI Assistant node.  
    - Edge Cases: Webhook connectivity issues, invalid or missing input data.  
    - Notes: Linked to https://www.npmjs.com/package/@n8n/chat for usage details.

#### 2.2 OpenAI Assistant Query with Vector Store

- **Overview:**  
  Sends the user input to an OpenAI assistant configured with a vector store for file retrieval, enabling the assistant to search documents and return responses with citations.

- **Nodes Involved:**  
  - OpenAI Assistant with Vector Store

- **Node Details:**  
  - **OpenAI Assistant with Vector Store**  
    - Type: `@n8n/n8n-nodes-langchain.openAi`  
    - Role: Queries the OpenAI assistant resource with vector store capabilities.  
    - Configuration: Uses a predefined assistant ID (`asst_QAfdobVCVCMJz8LmaEC7nlId`), disables preservation of original tools.  
    - Credentials: OpenAI API key configured (`OpenAi Lourival`).  
    - Inputs: From chat trigger node.  
    - Outputs: Connects to HTTP request node to retrieve thread content.  
    - Edge Cases: API authentication errors, assistant ID invalid, network timeouts.

#### 2.3 Thread Content Retrieval

- **Overview:**  
  Retrieves the entire conversation thread messages from OpenAI to ensure all citations are captured, as the assistant’s initial response may omit some citation details.

- **Nodes Involved:**  
  - Get ALL Thread Content  
  - Sticky Note2 (commentary)

- **Node Details:**  
  - **Get ALL Thread Content**  
    - Type: `n8n-nodes-base.httpRequest`  
    - Role: HTTP GET request to OpenAI API endpoint `/v1/threads/{{threadId}}/messages` to fetch all messages in the thread.  
    - Configuration: Uses OpenAI API credentials, sends header `OpenAI-Beta: assistants=v2`.  
    - Inputs: Receives threadId from assistant output.  
    - Outputs: Connects to splitting nodes for message processing.  
    - Edge Cases: API rate limits, invalid threadId, network errors.  
  - **Sticky Note2**  
    - Content: Explains necessity of retrieving all thread content due to incomplete citation retrieval by the assistant node.

#### 2.4 Data Splitting and Extraction

- **Overview:**  
  Processes the retrieved thread content by splitting it into individual messages, then splitting each message’s content and citations for granular handling.

- **Nodes Involved:**  
  - Split all message iterations from a thread  
  - Split all content from a single message  
  - Split all citations from a single message  
  - Sticky Note3, Sticky Note5 (comments)

- **Node Details:**  
  - **Split all message iterations from a thread**  
    - Type: `n8n-nodes-base.splitOut`  
    - Role: Splits the array of messages (`data` field) into individual items for processing.  
    - Inputs: From HTTP request node.  
    - Outputs: To split content node.  
    - Edge Cases: Empty or malformed data arrays.  
  - **Split all content from a single message**  
    - Type: `n8n-nodes-base.splitOut`  
    - Role: Splits the `content` field of each message into smaller parts if needed.  
    - Inputs: From previous split node.  
    - Outputs: To split citations node.  
    - Edge Cases: Missing or empty content fields.  
  - **Split all citations from a single message**  
    - Type: `n8n-nodes-base.splitOut`  
    - Role: Splits the `text.annotations` array containing citations within the message content.  
    - Inputs: From content split node.  
    - Outputs: To file metadata retrieval node.  
    - Edge Cases: Messages without annotations or empty arrays.  
  - **Sticky Note3**  
    - Content: Highlights that only `id`, `filename`, and `text` are needed from file retrieval requests.  
  - **Sticky Note5**  
    - Content: Explains aggregation necessity due to multiple citations and texts to substitute.

#### 2.5 Citation File Metadata Retrieval

- **Overview:**  
  For each citation, fetches the corresponding file metadata (filename) from OpenAI to enable proper citation referencing.

- **Nodes Involved:**  
  - Retrieve file name from a file ID

- **Node Details:**  
  - **Retrieve file name from a file ID**  
    - Type: `n8n-nodes-base.httpRequest`  
    - Role: HTTP GET request to OpenAI API `/v1/files/{{file_id}}` to get file details.  
    - Configuration: Uses OpenAI API credentials, query parameter `limit=1`.  
    - Inputs: Receives `file_citation.file_id` from citation split node.  
    - Outputs: To data regularization node.  
    - Error Handling: Configured to continue workflow on error (`onError: continueRegularOutput`).  
    - Edge Cases: Invalid file IDs, API errors, missing file metadata.

#### 2.6 Data Regularization and Aggregation

- **Overview:**  
  Normalizes the citation data to a consistent format and aggregates all citation entries into a single dataset for final processing.

- **Nodes Involved:**  
  - Regularize output  
  - Aggregate

- **Node Details:**  
  - **Regularize output**  
    - Type: `n8n-nodes-base.set`  
    - Role: Extracts and assigns key fields (`id`, `filename`, `text`) from the HTTP response to a normalized JSON structure.  
    - Inputs: From file metadata retrieval node.  
    - Outputs: To aggregate node.  
    - Edge Cases: Missing fields in input JSON.  
  - **Aggregate**  
    - Type: `n8n-nodes-base.aggregate`  
    - Role: Aggregates all normalized citation items into a single array for batch processing.  
    - Inputs: From regularize output node.  
    - Outputs: To final formatting code node.  
    - Edge Cases: Empty input arrays.

#### 2.7 Output Formatting

- **Overview:**  
  Replaces citation texts in the assistant’s output with formatted references using Markdown tags by default. Optionally converts Markdown to HTML.

- **Nodes Involved:**  
  - Finnaly format the output  
  - Optional Markdown to HTML  
  - Sticky Note6 (comment)

- **Node Details:**  
  - **Finnaly format the output**  
    - Type: `n8n-nodes-base.code`  
    - Role: Iterates over all citation data, replacing original citation texts in the assistant’s output with formatted strings referencing filenames in Markdown style (`_(filename)_`).  
    - Inputs: Aggregated citation data and original assistant output.  
    - Outputs: To optional Markdown to HTML node.  
    - Code snippet summary:  
      ```js
      let saida = $('OpenAI Assistant with Vector Store').item.json.output;
      for (let i of $input.item.json.data) {
        saida = saida.replaceAll(i.text, "  _(" + i.filename + ")_  ");
      }
      $input.item.json.output = saida;
      return $input.item;
      ```  
    - Edge Cases: Missing or empty output, citation text not found in output, multiple identical citations.  
  - **Optional Markdown to HTML** (disabled by default)  
    - Type: `n8n-nodes-base.markdown`  
    - Role: Converts the Markdown-formatted output into HTML if enabled.  
    - Inputs: From formatting code node.  
    - Outputs: Final output.  
    - Edge Cases: Disabled by default; enabling requires validation of Markdown content.  
  - **Sticky Note6**  
    - Content: Explains the purpose of the formatting code and the option to convert Markdown to HTML.

---

### 3. Summary Table

| Node Name                               | Node Type                                | Functional Role                                  | Input Node(s)                           | Output Node(s)                        | Sticky Note                                                                                              |
|----------------------------------------|-----------------------------------------|-------------------------------------------------|---------------------------------------|-------------------------------------|--------------------------------------------------------------------------------------------------------|
| Create a simple Trigger to have the Chat button within N8N | @n8n/n8n-nodes-langchain.chatTrigger   | Input reception via chat trigger                 | None                                  | OpenAI Assistant with Vector Store  | https://www.npmjs.com/package/@n8n/chat                                                                |
| OpenAI Assistant with Vector Store     | @n8n/n8n-nodes-langchain.openAi         | Query OpenAI assistant with vector store         | Create a simple Trigger                | Get ALL Thread Content              |                                                                                                        |
| Get ALL Thread Content                  | n8n-nodes-base.httpRequest               | Retrieve full conversation thread messages       | OpenAI Assistant with Vector Store    | Split all message iterations from a thread | ### Retrieving all thread content is necessary because the OpenAI tool does not retrieve all citations upon request. |
| Split all message iterations from a thread | n8n-nodes-base.splitOut                  | Split thread messages into individual messages   | Get ALL Thread Content                 | Split all content from a single message |                                                                                                        |
| Split all content from a single message | n8n-nodes-base.splitOut                  | Split message content                             | Split all message iterations from a thread | Split all citations from a single message |                                                                                                        |
| Split all citations from a single message | n8n-nodes-base.splitOut                  | Split citations within message content           | Split all content from a single message | Retrieve file name from a file ID   |                                                                                                        |
| Retrieve file name from a file ID       | n8n-nodes-base.httpRequest               | Fetch file metadata for citation                  | Split all citations from a single message | Regularize output                  |                                                                                                        |
| Regularize output                       | n8n-nodes-base.set                       | Normalize citation data fields                     | Retrieve file name from a file ID      | Aggregate                          | ### A file retrieval request contains a lot of information, and we want only the text that will be substituted and the file name. - id - filename - text |
| Aggregate                             | n8n-nodes-base.aggregate                 | Aggregate all citation data into one array        | Regularize output                      | Finnaly format the output           | ### With the last three splits, we may have many citations and texts to substitute. By doing an aggregation, it will be possible to handle everything as a single request. |
| Finnaly format the output              | n8n-nodes-base.code                      | Replace citation texts with formatted references  | Aggregate                             | Optional Markdown to HTML           | ### This simple code will take all the previous files and citations and alter the original text, formatting the output. In this way, we can use Markdown tags to create links, or if you prefer, we can add an HTML transformation node. |
| Optional Markdown to HTML              | n8n-nodes-base.markdown                  | Convert Markdown output to HTML (optional)        | Finnaly format the output              | None                              | Disabled by default                                                                                     |
| Window Buffer Memory                   | @n8n/n8n-nodes-langchain.memoryBufferWindow | AI memory buffer for context preservation          | None                                  | OpenAI Assistant with Vector Store |                                                                                                        |
| Sticky Note                           | n8n-nodes-base.stickyNote                | Documentation and description                      | None                                  | None                              | ## Make OpenAI Citation for File Retrieval RAG ... by Davi Saranszky Mesquita https://www.linkedin.com/in/mesquitadavi/ |
| Sticky Note1                          | n8n-nodes-base.stickyNote                | Setup instructions                                 | None                                  | None                              | ## Setup - Configure OpenAI Key ... assistant created within the OpenAI platform that contains a vector store |
| Sticky Note2                          | n8n-nodes-base.stickyNote                | Explanation of thread content retrieval necessity | None                                  | None                              | ### Retrieving all thread content is necessary because the OpenAI tool does not retrieve all citations upon request. |
| Sticky Note3                          | n8n-nodes-base.stickyNote                | Explanation of data fields needed                   | None                                  | None                              | ### A file retrieval request contains a lot of information, and we want only the text that will be substituted and the file name. - id - filename - text |
| Sticky Note4                          | n8n-nodes-base.stickyNote                | UI note about chat button                           | None                                  | None                              | ## Within N8N, there will be a chat button to test                                                            |
| Sticky Note5                          | n8n-nodes-base.stickyNote                | Explanation of aggregation necessity                | None                                  | None                              | ### With the last three splits, we may have many citations and texts to substitute. By doing an aggregation, it will be possible to handle everything as a single request. |
| Sticky Note6                          | n8n-nodes-base.stickyNote                | Explanation of output formatting code               | None                                  | None                              | ### This simple code will take all the previous files and citations and alter the original text, formatting the output. In this way, we can use Markdown tags to create links, or if you prefer, we can add an HTML transformation node. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Chat Trigger Node**  
   - Node Type: `@n8n/n8n-nodes-langchain.chatTrigger`  
   - Configuration: Default options, assign a webhook ID for external access.  
   - Purpose: To receive chat input inside n8n.

2. **Add OpenAI Assistant Node with Vector Store**  
   - Node Type: `@n8n/n8n-nodes-langchain.openAi`  
   - Configuration:  
     - Resource: `assistant`  
     - Assistant ID: Use your OpenAI assistant ID with vector store enabled (e.g., `asst_QAfdobVCVCMJz8LmaEC7nlId`)  
     - Options: Disable "preserveOriginalTools"  
   - Credentials: Configure OpenAI API key credentials.  
   - Connect input from Chat Trigger node.

3. **Add HTTP Request Node to Get Full Thread Content**  
   - Node Type: `n8n-nodes-base.httpRequest`  
   - Configuration:  
     - Method: GET  
     - URL: `https://api.openai.com/v1/threads/{{ $json.threadId }}/messages` (use expression to inject threadId)  
     - Headers: Add header `OpenAI-Beta: assistants=v2`  
     - Authentication: Use OpenAI API credentials  
   - Connect input from OpenAI Assistant node.

4. **Add Split Node to Split Thread Messages**  
   - Node Type: `n8n-nodes-base.splitOut`  
   - Configuration:  
     - Field to split out: `data` (array of messages)  
   - Connect input from HTTP Request node.

5. **Add Split Node to Split Message Content**  
   - Node Type: `n8n-nodes-base.splitOut`  
   - Configuration:  
     - Field to split out: `content`  
   - Connect input from previous split node.

6. **Add Split Node to Split Citations in Message**  
   - Node Type: `n8n-nodes-base.splitOut`  
   - Configuration:  
     - Field to split out: `text.annotations` (array of citations)  
   - Connect input from previous split node.

7. **Add HTTP Request Node to Retrieve File Metadata**  
   - Node Type: `n8n-nodes-base.httpRequest`  
   - Configuration:  
     - Method: GET  
     - URL: `https://api.openai.com/v1/files/{{ $json.file_citation.file_id }}` (expression to inject file ID)  
     - Query Parameter: `limit=1`  
     - Authentication: Use OpenAI API credentials  
     - On Error: Continue workflow (to handle missing files gracefully)  
   - Connect input from citation split node.

8. **Add Set Node to Regularize Output**  
   - Node Type: `n8n-nodes-base.set`  
   - Configuration: Assign fields:  
     - `id` = `{{$json.id}}`  
     - `filename` = `{{$json.filename}}` (from HTTP response)  
     - `text` = `{{$('Split all citations from a single message').item.json.text}}` (citation text)  
   - Connect input from file metadata HTTP request node.

9. **Add Aggregate Node to Combine Citation Data**  
   - Node Type: `n8n-nodes-base.aggregate`  
   - Configuration: Aggregate all items into one array.  
   - Connect input from Set node.

10. **Add Code Node to Format Final Output**  
    - Node Type: `n8n-nodes-base.code`  
    - Configuration: Use JavaScript code to replace citation texts in assistant output with formatted references:  
      ```js
      let saida = $('OpenAI Assistant with Vector Store').item.json.output;
      for (let i of $input.item.json.data) {
        saida = saida.replaceAll(i.text, "  _(" + i.filename + ")_  ");
      }
      $input.item.json.output = saida;
      return $input.item;
      ```  
    - Connect input from Aggregate node.

11. **(Optional) Add Markdown Node to Convert to HTML**  
    - Node Type: `n8n-nodes-base.markdown`  
    - Configuration:  
      - Input field: `output` (from previous code node)  
      - Destination key: `output`  
    - Disabled by default; enable if HTML output is desired.  
    - Connect input from Code node.

12. **Add Window Buffer Memory Node (Optional for Context)**  
    - Node Type: `@n8n/n8n-nodes-langchain.memoryBufferWindow`  
    - Configuration: Default.  
    - Connect output to OpenAI Assistant node’s AI memory input.

13. **Add Sticky Notes for Documentation**  
    - Add sticky notes at appropriate positions with content from the original workflow for clarity and instructions.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                   | Context or Link                                                                                   |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Workflow created by Davi Saranszky Mesquita.                                                                                                                                                                                   | https://www.linkedin.com/in/mesquitadavi/                                                      |
| The workflow uses an OpenAI assistant with vector store capabilities to perform file retrieval augmented generation (RAG) with citations.                                                                                     | Workflow description and use case.                                                              |
| Chat trigger node enables testing the assistant directly within n8n via a chat button.                                                                                                                                         | https://www.npmjs.com/package/@n8n/chat                                                        |
| Retrieving all thread content is necessary because the OpenAI assistant node does not return all citations in the initial response.                                                                                           | Sticky Note2 content.                                                                           |
| The final formatting code replaces citation texts with Markdown links referencing the source filenames; optionally, this can be converted to HTML.                                                                            | Sticky Note6 content.                                                                           |
| The workflow requires an OpenAI API key configured in n8n credentials with appropriate permissions for assistant and file retrieval APIs.                                                                                     | Setup instructions.                                                                            |
| The workflow is designed to be extensible; users can customize the final formatting code to adjust citation styles or add additional output transformations (e.g., HTML conversion).                                            | Customization note in description.                                                             |

---

This documentation provides a complete, structured understanding of the workflow, enabling reproduction, modification, and troubleshooting by advanced users or automation agents.