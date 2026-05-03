Convert Documents to Markdown with MinerU API and GPT-4o-mini

https://n8nworkflows.xyz/workflows/convert-documents-to-markdown-with-mineru-api-and-gpt-4o-mini-4808


# Convert Documents to Markdown with MinerU API and GPT-4o-mini

---
### 1. Workflow Overview

This workflow, titled **"Convert Documents to Markdown with MinerU API and GPT-4o-mini"**, is designed to process user-uploaded documents via a chat-triggered interface, convert these documents into a Markdown format, and enable AI-driven interaction and processing over the document content. It targets scenarios where users upload multiple files via chat, and an AI agent parses and processes these files using both the MinerU MCP API and OpenAI's GPT-4o-mini model.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception and Initialization:** Receives chat messages with file uploads and initializes tracking variables.
- **1.2 File Iteration and Local Storage:** Iterates over each uploaded file, saves it locally, and tracks processed files.
- **1.3 AI Processing:** Uses the MinerU MCP API and GPT-4o-mini OpenAI model in tandem via an AI Agent node to analyze and respond based on the uploaded documents.
- **1.4 Loop Control:** Checks whether all files have been processed, controlling the iteration flow.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and Initialization

- **Overview:**  
  This block triggers on receiving a chat message that may contain file uploads. It initializes an empty array to hold file information and sets an index to zero for iteration.

- **Nodes Involved:**  
  - When chat message received  
  - 初始化files (Initialize files)  
  - If 还没遍历结束 (If iteration not finished)

- **Node Details:**

  - **When chat message received**  
    - *Type:* Chat Trigger node, webhook-based entry point  
    - *Role:* Listens for incoming chat messages and allows file uploads.  
    - *Configuration:* Accepts all MIME types for uploaded files (`allowedFilesMimeTypes: "*"`).  
    - *Input/Output:* No prior input; outputs the chat message data including uploaded files in binary form.  
    - *Edge Cases:* Possible issues include large file uploads causing timeouts or unsupported file types despite wildcard MIME acceptance.  
    - *Notes:* Acts as the workflow’s entry point.

  - **初始化files (Initialize files)**  
    - *Type:* Set node  
    - *Role:* Initializes two variables: an empty array `files` to store file paths/names and an integer `index` at 0 for tracking iteration.  
    - *Configuration:* Sets `files = []`, `index = 0` and retains other incoming data fields.  
    - *Input:* Receives output from the chat trigger node  
    - *Output:* Passes initialized variables downstream.

  - **If 还没遍历结束 (If iteration not finished)**  
    - *Type:* If node  
    - *Role:* Evaluates if the current file index is less than the total number of uploaded files to control loop continuation.  
    - *Configuration:*  
      - Condition: `{{$json.index}} < {{$('When chat message received').item.binary.values().length}}`  
      - This ensures iteration continues until all files are processed.  
    - *Input:* From 初始化files or loop returns  
    - *Outputs:*  
      - True branch triggers processing of the current file  
      - False branch ends iteration and proceeds to AI processing.  
    - *Edge Cases:* Failures if the number of uploaded files changes during processing or if index is not properly incremented.

---

#### 1.2 File Iteration and Local Storage

- **Overview:**  
  This block handles the actual processing of each uploaded file by saving them locally and updating the list of processed files and the index.

- **Nodes Involved:**  
  - 文件保存到本地 (Save file locally)  
  - 添加文件路径到 (Add file path to list)  
  - If 还没遍历结束 (If iteration not finished) [loop back]

- **Node Details:**

  - **文件保存到本地 (Save file locally)**  
    - *Type:* Read/Write File node  
    - *Role:* Writes each uploaded file locally to a specified folder with a unique timestamped filename.  
    - *Configuration:*  
      - Operation: Write  
      - FileName: `/Users/adrianwang/Documents/ailab/minerU-mcp/downloads/file_{{$now.toMillis()}}.{{ extension of current file }}`  
      - Data source: Uses data from the current file in the binary data array indexed by `index`.  
    - *Input:* From If node’s true branch  
    - *Output:* Passes data to the next node to update file list.  
    - *Edge Cases:* File write permission errors, disk space issues, filename collisions if run rapidly.

  - **添加文件路径到 (Add file path to list)**  
    - *Type:* Set node  
    - *Role:* Appends the newly saved file’s path to the `files` array and increments the `index` by 1 for next iteration.  
    - *Configuration:*  
      - `files = $json.files.append($json.fileName)`  
      - `index = $json.index + 1`  
    - *Input:* Receives output from 文件保存到本地  
    - *Output:* Loops back to the If node to check if more files remain.  
    - *Edge Cases:* Potential errors if `files` is undefined or improper append operations.

  - **If 还没遍历结束 (If iteration not finished)**  
    - *As above, loops control mechanism.*

---

#### 1.3 AI Processing

- **Overview:**  
  After all files are saved and recorded, this block invokes the AI Agent that integrates MinerU MCP API and OpenAI’s GPT-4o-mini model to analyze the uploaded documents and respond to the user's chat input.

- **Nodes Involved:**  
  - AI Agent  
  - MCP Client  
  - OpenAI Chat Model

- **Node Details:**

  - **AI Agent**  
    - *Type:* LangChain Agent node  
    - *Role:* Central AI orchestrator that uses system and user prompts, integrating document data for intelligent responses.  
    - *Configuration:*  
      - `text` input interpolates user chat input:  
        ```
        ---  
        以下为用户的对话信息：  
        {{ $('When chat message received').item.json.chatInput }}  
        ```  
      - `systemMessage` guides the agent to use file parsing tools when requested and includes uploaded file list:  
        ```
        你是一个具备文件阅读能力的AI，用户如果请求对于文件进行处理，就请使用系统中已有的文档解析工具进行解析后，按照用户要求进行处理  
        ---  
        下面为用户上传的文件列表：  
        {{ $json.files }}
        ```  
      - `promptType`: define  
    - *Inputs:* Receives AI tool and language model responses.  
    - *Outputs:* Final AI response to the chat.  
    - *Edge Cases:* Prompt construction errors, empty or malformed files array, API rate limits, or service downtime.

  - **MCP Client**  
    - *Type:* LangChain MCP Client Tool node  
    - *Role:* Connects to MinerU MCP API for document parsing and processing.  
    - *Configuration:*  
      - `sseEndpoint` set to `http://127.0.0.1:8001/sse` (local service endpoint).  
      - `include`: selected (only selected outputs included).  
    - *Input:* AI Agent uses this as an AI tool.  
    - *Edge Cases:* Connection errors if local MinerU MCP service is not running, invalid endpoint, SSE stream interruptions.

  - **OpenAI Chat Model**  
    - *Type:* LangChain OpenAI Chat Model node  
    - *Role:* Provides GPT-4o-mini model responses.  
    - *Configuration:*  
      - Model: `gpt-4o-mini`  
      - Credentials: OpenAI API key provided (`ailab` credential).  
      - No additional options configured.  
    - *Input:* AI Agent uses this as the language model.  
    - *Edge Cases:* API key invalid, rate limits, model unavailability.

---

#### 1.4 Loop Control

- **Overview:**  
  This block is essentially the `If 还没遍历结束` node reused for loop control — it determines whether to continue processing files or proceed to AI processing.

- **Nodes Involved:**  
  - If 还没遍历结束 (If iteration not finished) — used both at the start of iteration and after each file is processed.

---

### 3. Summary Table

| Node Name               | Node Type                                  | Functional Role                                  | Input Node(s)               | Output Node(s)              | Sticky Note                                 |
|-------------------------|--------------------------------------------|-------------------------------------------------|-----------------------------|-----------------------------|---------------------------------------------|
| When chat message received | LangChain Chat Trigger                     | Entry point for chat messages and file uploads  | None                        | 初始化files                  |                                             |
| 初始化files (Initialize files) | Set                                       | Initialize empty files array and index          | When chat message received  | If 还没遍历结束             |                                             |
| If 还没遍历结束 (If iteration not finished) | If                                        | Loop control: check if more files need processing | 初始化files, 添加文件路径到 | 文件保存到本地, AI Agent     |                                             |
| 文件保存到本地 (Save file locally) | Read/Write File                            | Save each uploaded file locally                   | If 还没遍历结束 (true branch) | 添加文件路径到              |                                             |
| 添加文件路径到 (Add file path to list) | Set                                       | Append saved file path to list and increment index | 文件保存到本地              | If 还没遍历结束             |                                             |
| AI Agent                 | LangChain Agent                            | Core AI orchestrator integrating document tools | MCP Client, OpenAI Chat Model | Output final AI response    |                                             |
| MCP Client               | LangChain MCP Client Tool                  | Connects to MinerU MCP API for document parsing | AI Agent                    | AI Agent                    |                                             |
| OpenAI Chat Model        | LangChain OpenAI Chat Model                | Provides GPT-4o-mini language model response    | AI Agent                    | AI Agent                    | Credential “ailab” used for OpenAI API       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the chat message trigger node:**  
   - Type: LangChain Chat Trigger  
   - Configure webhook with file uploads allowed (`allowFileUploads: true`)  
   - Accept all MIME types (`allowedFilesMimeTypes: "*"`)  
   - Position it as the workflow entry point.

2. **Add a Set node named “初始化files” (Initialize files):**  
   - Set fields:  
     - `files` = empty array `[]`  
     - `index` = number `0`  
   - Enable "Include Other Fields" to pass the input data through.  
   - Connect from the chat trigger node.

3. **Add an If node named “If 还没遍历结束” (If iteration not finished):**  
   - Condition: `{{$json.index}} < {{$('When chat message received').item.binary.values().length}}`  
   - Use version 2 condition type.  
   - Connect from “初始化files” node (and later loop back from “添加文件路径到”).

4. **Add Read/Write File node named “文件保存到本地” (Save file locally):**  
   - Operation: Write  
   - File name: `/Users/adrianwang/Documents/ailab/minerU-mcp/downloads/file_{{$now.toMillis()}}.{{ $('When chat message received').item.binary.values()[$json.index].fileName.split('.').pop() }}`  
   - Data Property Name: `=data{{$json.index}}` (access binary data of current index)  
   - Connect from the If node’s true branch.

5. **Add Set node named “添加文件路径到” (Add file path to list):**  
   - Assignments:  
     - `files` = append current fileName to existing files array: `{{$json.files.append($json.fileName)}}`  
     - `index` = increment current index by 1: `{{$json.index + 1}}`  
   - Enable "Include Other Fields."  
   - Connect from “文件保存到本地.”

6. **Loop back:**  
   - Connect “添加文件路径到” node output back to the “If 还没遍历结束” node input for re-evaluation.

7. **Add MCP Client node:**  
   - Type: LangChain MCP Client Tool  
   - Parameter: `sseEndpoint` set to `http://127.0.0.1:8001/sse`  
   - Include outputs: selected  
   - Position downstream.

8. **Add OpenAI Chat Model node:**  
   - Type: LangChain OpenAI Chat Model  
   - Model: `gpt-4o-mini`  
   - Credential: OpenAI API key configured under credential name “ailab”  
   - Position downstream.

9. **Add AI Agent node:**  
   - Type: LangChain Agent  
   - Parameters:  
     - `text` parameter:  
       ```
       ---  
       以下为用户的对话信息：  
       {{ $('When chat message received').item.json.chatInput }}  
       ```  
     - `systemMessage` parameter:  
       ```
       你是一个具备文件阅读能力的AI，用户如果请求对于文件进行处理，就请使用系统中已有的文档解析工具进行解析后，按照用户要求进行处理  
       ---  
       下面为用户上传的文件列表：  
       {{ $json.files }}
       ```  
     - `promptType`: define  
   - Connect two inputs from MCP Client (as AI tool) and OpenAI Chat Model (as language model).  
   - Connect AI Agent input from the false branch of the If node (when all files processed).

10. **Configure workflow execution order:**  
    - Ensure that the loop nodes execute sequentially and that AI processing only begins after all files are processed.

---

### 5. General Notes & Resources

| Note Content                                                                                  | Context or Link                                                  |
|-----------------------------------------------------------------------------------------------|-----------------------------------------------------------------|
| The MinerU MCP API endpoint is local (`http://127.0.0.1:8001/sse`); ensure MinerU MCP service is running locally. | MCP Client node configuration                                    |
| OpenAI credential named “ailab” must be pre-configured with valid API key for GPT-4o-mini usage. | OpenAI Chat Model node                                           |
| The workflow uses LangChain integration nodes for advanced AI orchestration within n8n.        | General workflow architecture                                   |
| The filename for saved files uses a timestamp to avoid collisions: `file_{{$now.toMillis()}}`  | 文件保存到本地 node configuration                               |
| The workflow can handle multiple file uploads per chat message and process them sequentially.  | Workflow input reception and loop control                       |

---

This documentation fully covers the workflow's structure, nodes, parameters, and logic, enabling reproduction, modification, and troubleshooting for advanced users and automation agents.