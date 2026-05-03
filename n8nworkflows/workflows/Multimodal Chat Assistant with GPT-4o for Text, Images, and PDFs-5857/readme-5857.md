Multimodal Chat Assistant with GPT-4o for Text, Images, and PDFs

https://n8nworkflows.xyz/workflows/multimodal-chat-assistant-with-gpt-4o-for-text--images--and-pdfs-5857


# Multimodal Chat Assistant with GPT-4o for Text, Images, and PDFs

### 1. Workflow Overview

This workflow implements a **Multimodal Chat Assistant** leveraging OpenAI's GPT-4o model, designed to interact with users via text, images, and PDFs. It allows users to upload files or input text queries and receive AI-generated responses that consider both the uploaded contents and conversational context.

The workflow is structured into the following logical blocks:

- **1.1 Input Reception and Routing:**  
  Handles incoming chat messages and determines if they contain files (images/PDFs) or only text, routing accordingly.

- **1.2 File Analysis and Content Summarization:**  
  Processes uploaded images or PDFs using GPT-4o’s multimodal capabilities to generate detailed descriptions and suggest potential questions based on the content.

- **1.3 Text Question Handling and Memory Management:**  
  Manages conversational state with a memory buffer, processes user text queries with context, and generates AI responses using advanced language models.

- **1.4 AI Agent Processing:**  
  Employs an AI agent node to provide expert-level answers by combining user queries with the analyzed content and chat history.

- **1.5 Memory Buffer and Message Management:**  
  Maintains a session-based memory buffer to preserve conversational context and supports smooth, contextual multi-turn interactions.

---

### 2. Block-by-Block Analysis

---

#### 2.1 Input Reception and Routing

**Overview:**  
This block receives incoming chat messages, including uploaded files, and routes the workflow based on whether a file is present.

**Nodes Involved:**  
- `chat` (Chat Trigger)  
- `If` (Conditional Node)

**Node Details:**

- **chat**  
  - *Type:* Chat Trigger  
  - *Role:* Entry point of the workflow, receives user messages and allows file uploads.  
  - *Configuration:*  
    - Public webhook enabled to allow external access.  
    - File uploads allowed.  
  - *Input/Output:* No input; outputs chat data with possible files.  
  - *Potential Failures:* Webhook availability, file size limits, malformed requests.

- **If**  
  - *Type:* If (Conditional)  
  - *Role:* Checks if the incoming message contains an uploaded file.  
  - *Configuration:* Condition checks if the first file's filename is not empty (`{{$json.files[0].fileName}}`).  
  - *Input:* Output from `chat` node.  
  - *Output:*  
    - True branch: File present -> routed to file analysis.  
    - False branch: No file -> routed to text processing.  
  - *Potential Failures:* Array index out of range if files array missing; expression evaluation errors.

---

#### 2.2 File Analysis and Content Summarization

**Overview:**  
When a file is attached (image or PDF), this block analyzes the content using the GPT-4o multimodal model to generate a detailed description and suggest three questions the user might ask.

**Nodes Involved:**  
- `OpenAI` (OpenAI Multimodal Analysis)  
- `Basic LLM Chain` (Text Summarization and Question Suggestion)  
- `chatmem` (Memory Manager for File Content)  

**Node Details:**

- **OpenAI**  
  - *Type:* OpenAI Langchain Node (Multimodal)  
  - *Role:* Analyzes image or PDF file content in base64 format.  
  - *Configuration:*  
    - Model: `gpt-4o` multimodal capable.  
    - Operation: `analyze` resource type `image` (also handles PDFs).  
    - Input type: base64 data from binary property `data0`.  
    - Option: High detail description requested.  
    - Prompt: Describes content and suggests 3 possible user questions.  
  - *Input:* Binary data of uploaded file from `If` node’s true branch.  
  - *Output:* Generated detailed description with suggested questions.  
  - *Potential Failures:* API auth errors, unsupported file types, large file sizes causing timeouts.

- **Basic LLM Chain**  
  - *Type:* Langchain LLM Chain  
  - *Role:* Further processes the description to focus on chat input context.  
  - *Configuration:*  
    - Prompt uses both the content description and user chat input to tailor response.  
  - *Input:* Output from `OpenAI` node and current chat text.  
  - *Output:* Refined descriptive text used for conversation memory.  
  - *Potential Failures:* Expression errors if input JSON paths invalid.

- **chatmem**  
  - *Type:* Memory Manager  
  - *Role:* Inserts the analyzed content text into conversation memory under the AI role.  
  - *Configuration:* Mode `insert`; adds AI message `{{ $json.content }}`.  
  - *Input:* Output from `Basic LLM Chain`.  
  - *Output:* Updated memory state.  
  - *Potential Failures:* Memory key misconfiguration, session key issues.

---

#### 2.3 Text Question Handling and Memory Management

**Overview:**  
This block handles text-only user queries, manages session-based memory buffers, and prepares data for AI response generation.

**Nodes Involved:**  
- `chatmem1` (Memory Manager for Text)  
- `AI Agent` (Expert AI processing)  
- `Simple Memory1` (Memory Buffer for file content)  
- `Simple Memory2` (Memory Buffer for AI Agent conversation)  
- `Simple Memory` (Memory Buffer for AI Agent input)  
- `OpenAI Chat Model1` (Language Model for AI Agent)  
- `OpenAI Chat Model` (Language Model for Basic LLM Chain)

**Node Details:**

- **chatmem1**  
  - *Type:* Memory Manager  
  - *Role:* Manages memory messages for text queries, no simplify output.  
  - *Input:* False branch output from `If`.  
  - *Output:* Passed to `AI Agent`.  
  - *Potential Failures:* Session key mismatch, memory overflow.

- **AI Agent**  
  - *Type:* Langchain Agent  
  - *Role:* Processes user queries with expert knowledge, considering chat input and last AI content.  
  - *Configuration:*  
    - Prompt includes user query and last AI message content.  
  - *Input:* Output from `chatmem1` and `Simple Memory` nodes.  
  - *Output:* AI-generated response.  
  - *Potential Failures:* Model rate limits, prompt formatting errors.

- **Simple Memory1 / Simple Memory2 / Simple Memory**  
  - *Type:* Memory Buffer Window  
  - *Role:* Maintain session-based conversational memory with a sliding window.  
  - *Configuration:* Session key derived from `sessionId` in chat JSON.  
  - *Input/Output:* Connected to memory manager nodes and AI Agent for context persistence.  
  - *Potential Failures:* Session key collisions, memory size limits causing truncation.

- **OpenAI Chat Model & OpenAI Chat Model1**  
  - *Type:* Langchain LM Chat OpenAI  
  - *Role:* Provide GPT-4o language model capabilities for text generation in chat and AI agent nodes.  
  - *Configuration:* Using GPT-4o with default options.  
  - *Input/Output:* Linked as language model input to downstream nodes.  
  - *Potential Failures:* API key issues, request timeouts.

---

#### 2.4 AI Agent Processing

**Overview:**  
This block represents the core AI response generator, combining user inputs, previous conversation, and file content to deliver expert answers.

**Nodes Involved:**  
- `AI Agent`  
- `OpenAI Chat Model1`  
- `Simple Memory`  
- `chatmem1`

**Node Details:**

- **AI Agent**  
  - As detailed above: uses the latest user input and prior AI responses stored in memory to generate contextually rich answers.  
  - Inputs from memory buffer and chat memory.  
  - Edge cases: prompt injection, malformed user input.

- **OpenAI Chat Model1**  
  - Provides the language model capability required by the AI Agent, configured for GPT-4o.

- **Simple Memory**  
  - Supplies the AI Agent with session-specific memory to maintain conversational context.

- **chatmem1**  
  - Manages memory insertions and retrievals related to the chat for AI Agent consumption.

---

### 3. Summary Table

| Node Name           | Node Type                                   | Functional Role                                  | Input Node(s)           | Output Node(s)              | Sticky Note                                                                                      |
|---------------------|---------------------------------------------|-------------------------------------------------|-------------------------|-----------------------------|------------------------------------------------------------------------------------------------|
| chat                | Chat Trigger                                | Entry point, receives chat messages and files   | -                       | If                          |                                                                                                |
| If                  | Conditional                                 | Routes based on presence of attached file       | chat                    | OpenAI (True branch), chatmem1 (False branch) | This route runs when an image/file/pdf is attached. It saves what it sees in the chatmem         |
| OpenAI              | Langchain OpenAI (Multimodal Analyze)      | Analyzes images/PDFs to describe content         | If                      | Basic LLM Chain              |                                                                                                |
| Basic LLM Chain      | Langchain LLM Chain                         | Refines content description and suggests questions | OpenAI                   | chatmem                     |                                                                                                |
| chatmem             | Memory Manager                              | Inserts analyzed content into conversation memory | Basic LLM Chain           | -                           |                                                                                                |
| chatmem1            | Memory Manager                              | Manages memory for text-only chat input          | If                       | AI Agent                    | This runs after the upload, it retrieves the initial content from the memory                     |
| AI Agent            | Langchain Agent                             | Generates expert AI responses                      | chatmem1, Simple Memory   | -                           |                                                                                                |
| Simple Memory1       | Memory Buffer Window                        | Maintains session memory for file content         | -                       | chatmem                     |                                                                                                |
| Simple Memory2       | Memory Buffer Window                        | Maintains session memory for AI Agent             | -                       | chatmem1                    |                                                                                                |
| Simple Memory        | Memory Buffer Window                        | Provides memory context to AI Agent                | -                       | AI Agent                    |                                                                                                |
| OpenAI Chat Model    | Langchain LM Chat OpenAI                    | Provides GPT-4o model for Basic LLM Chain         | -                       | Basic LLM Chain             |                                                                                                |
| OpenAI Chat Model1   | Langchain LM Chat OpenAI                    | Provides GPT-4o model for AI Agent                 | -                       | AI Agent                   |                                                                                                |
| Sticky Note          | Sticky Note                                | Describes overall workflow purpose                 | -                       | -                           | ## Chat with thing\n\nThis n8n template lets you build a smart AI chat assistant that can handle text, images, and PDFs — using OpenAI's GPT-4o multimodal model. It supports dynamic conversations and file analysis, making it great for AI-driven support bots, personal assistants, or embedded chat widgets. |
| Sticky Note1         | Sticky Note                                | Notes file upload routing                           | -                       | -                           | This route runs when an image/file/pdf is attached. It saves what it sees in the chatmem          |
| Sticky Note2         | Sticky Note                                | Notes conversation setup                           | -                       | -                           | Use this node to set the conversation                                                             |
| Sticky Note3         | Sticky Note                                | Notes retrieval of initial content from memory    | -                       | -                           | This runs after the upload, it retrieves the initial content from the memory                      |
| Sticky Note4         | Sticky Note                                | Notes main chat prompt customization               | -                       | -                           | This is the main chat, adjust the prompt to match your use case.                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Chat Trigger Node (`chat`)**  
   - Type: Langchain Chat Trigger  
   - Configure webhook access as public.  
   - Enable file uploads.  
   - Leave initial messages empty.  
   - Position: Entry point.

2. **Add Conditional Node (`If`)**  
   - Type: If  
   - Condition: Check if `{{$json.files[0].fileName}}` is not empty.  
   - Input: Connect from `chat` node output.  
   - Outputs: True branch for file handling; False branch for text-only.

3. **Create OpenAI Multimodal Analysis Node (`OpenAI`)**  
   - Type: Langchain OpenAI Node (OpenAI)  
   - Model: `gpt-4o`  
   - Operation: Analyze resource type `image` (supports PDFs)  
   - Input Type: base64 binary property `data0` (connect binary data from file upload)  
   - Text prompt: "Describe the content of the image or pdf in detail then wait for questions about it. Based on what is in the content, suggest 3 questions the user may ask"  
   - Credentials: Link your OpenAI API key.  
   - Connect True output of `If` node to this node.

4. **Add Basic LLM Chain Node (`Basic LLM Chain`)**  
   - Type: Langchain LLM Chain  
   - Prompt: Use expression:  
     ```
     Describe `{{ $json.content }}`

     Use the text from the chat to focus the response:  
     `{{ $('chat').item.json.chatInput }}`
     ```  
   - Model: Connected to OpenAI Chat Model (step 7)  
   - Input: Connect from `OpenAI` node output.

5. **Add Memory Manager Node (`chatmem`)**  
   - Type: Langchain Memory Manager  
   - Mode: Insert  
   - Message: AI role message with content from `Basic LLM Chain` output `{{$json.content}}`  
   - Input: Connect from `Basic LLM Chain`.

6. **Create Memory Buffer Window Nodes**  
   - `Simple Memory1`:  
     - Session Key: `{{$json.sessionId}}` (custom key)  
     - Purpose: Buffer for file content memory  
   - `Simple Memory2`:  
     - Session Key: `{{$json.sessionId}}` (custom key)  
     - Purpose: Buffer for AI Agent conversation  
   - `Simple Memory`:  
     - Session Key: `{{$json.sessionId}}` (custom key)  
     - Purpose: Buffer for AI Agent input  
   - Connect relevant inputs/outputs according to memory flow.

7. **Add OpenAI Chat Model Nodes**  
   - `OpenAI Chat Model`:  
     - Model: `gpt-4o`  
     - Used by `Basic LLM Chain`  
   - `OpenAI Chat Model1`:  
     - Model: `gpt-4o`  
     - Used by `AI Agent`

8. **Add Memory Manager Node for Text Input (`chatmem1`)**  
   - Type: Langchain Memory Manager  
   - Mode: Default (managing conversation for text input)  
   - Simplify Output: False  
   - Input: Connect from False branch of `If` node (text only).

9. **Add AI Agent Node (`AI Agent`)**  
   - Type: Langchain Agent  
   - Prompt:  
     ```
     You are an expert and will help the user with their query `{{ $("chat").item.json.chatInput }}` about  
     {{ $json.messages[$json.messages.length - 1].kwargs.content }}
     ```  
   - Model: Use `OpenAI Chat Model1`  
   - Input: Connect from `chatmem1` and `Simple Memory` nodes.  
   - Output: Final AI response.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                      | Context or Link                                                                                                  |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| This n8n template lets you build a smart AI chat assistant that can handle text, images, and PDFs — using OpenAI's GPT-4o multimodal model. It supports dynamic conversations and file analysis, making it great for AI-driven support bots, personal assistants, or embedded chat widgets. | Sticky Note in workflow describing overall purpose.                                                            |
| This route runs when an image/file/pdf is attached. It saves what it sees in the chatmem.                                                                                                                                         | Sticky Note for file upload routing block.                                                                       |
| Use this node to set the conversation.                                                                                                                                                                                           | Sticky Note near `Basic LLM Chain` node.                                                                         |
| This runs after the upload, it retrieves the initial content from the memory.                                                                                                                                                      | Sticky Note near `chatmem1` node.                                                                                 |
| This is the main chat, adjust the prompt to match your use case.                                                                                                                                                                  | Sticky Note near the `AI Agent` node.                                                                            |

---

**Disclaimer:** The provided text is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to applicable content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.