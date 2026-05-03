Process Legal Documents with Ollama AI & Generate HTML Reports (100% Local)

https://n8nworkflows.xyz/workflows/process-legal-documents-with-ollama-ai---generate-html-reports--100--local--4869


# Process Legal Documents with Ollama AI & Generate HTML Reports (100% Local)

---
  
### 1. Workflow Overview

This workflow automates the processing of legal documents stored locally as PDF or other file types. Upon detecting a new or updated local file, it extracts the content, processes it via an AI agent (leveraging a local Ollama AI model integrated through LangChain), and generates an HTML report saved back to disk. The entire process is designed to operate 100% locally, ensuring privacy and security of sensitive legal documents.

The workflow is logically divided into these blocks:

- **1.1 Local File Detection & Reading:** Watches for new or modified files locally and reads their content.
- **1.2 Content Extraction:** Extracts readable text/content from the file.
- **1.3 AI Processing:** Sends the extracted content to an AI agent for analysis or summarization using a local OpenAI-compatible chat model.
- **1.4 Report Generation & Saving:** Converts the AI output to an HTML report file and writes it back to disk.

---

### 2. Block-by-Block Analysis

#### 2.1 Local File Detection & Reading

- **Overview:**  
  Detects new or changed files on the local system and reads their contents for further processing.

- **Nodes Involved:**  
  - Local File Trigger  
  - Read/Write Files from Disk

- **Node Details:**

  - **Local File Trigger**  
    - *Type & Role:* Event trigger node that monitors a specified local folder for file changes or additions.  
    - *Configuration:* Default parameters used (folder path and filters presumably set at runtime or via credentials).  
    - *Expressions/Variables:* None explicitly configured.  
    - *Connections:* Output connected to "Read/Write Files from Disk".  
    - *Edge Cases:*  
      - Missing or inaccessible folder path leads to trigger failure.  
      - File system permission errors.  
      - Rapid file changes may cause missed events or duplicate triggers.  
    - *Version Specific:* Requires n8n supporting local file system triggers.

  - **Read/Write Files from Disk**  
    - *Type & Role:* Reads raw file data from the file path provided by the trigger.  
    - *Configuration:* Reads incoming file path and loads file content into the workflow data.  
    - *Expressions:* Reads binary or text content depending on the file type.  
    - *Connections:* Input from "Local File Trigger"; output to "Extract from File".  
    - *Edge Cases:*  
      - File locks or access errors.  
      - Unsupported file formats or corrupted files.  
    - *Version Specific:* Standard n8n node.

---

#### 2.2 Content Extraction

- **Overview:**  
  Extracts text or structured data from the raw file content to prepare it for AI analysis.

- **Nodes Involved:**  
  - Extract from File

- **Node Details:**

  - **Extract from File**  
    - *Type & Role:* Extractor node that parses file content (PDF, DOCX, etc.) into plain text or structured format.  
    - *Configuration:* Default extraction parameters, likely auto-detecting file type and extracting text accordingly.  
    - *Expressions:* None explicitly; operates on input binary or text file data.  
    - *Connections:* Input from "Read/Write Files from Disk"; output to "AI Agent".  
    - *Edge Cases:*  
      - Unsupported file types or encrypted PDFs may cause extraction failures.  
      - Large files may cause timeouts or memory issues.  
    - *Version Specific:* Requires n8n version supporting this node (v1).  
    - *Notes:* Extraction accuracy depends on file quality and format.

---

#### 2.3 AI Processing

- **Overview:**  
  Processes the extracted text using an AI agent configured with a local OpenAI-compatible language model to analyze, summarize, or transform the content.

- **Nodes Involved:**  
  - AI Agent  
  - OpenAI Chat Model

- **Node Details:**

  - **AI Agent**  
    - *Type & Role:* LangChain agent node that orchestrates AI tasks using configured language models and tools.  
    - *Configuration:* Uses the "OpenAI Chat Model" as its language model input. Task details and prompts are configured inside the node or passed dynamically.  
    - *Expressions:* May use workflow expressions to pass extracted text as input prompt.  
    - *Connections:* Input from "Extract from File"; AI language model input connected to "OpenAI Chat Model"; output to "Convert to File".  
    - *Edge Cases:*  
      - Model loading errors.  
      - Unexpected or malformed input causing prompt failures.  
      - Timeout or performance issues with local AI model.  
    - *Version Specific:* Requires LangChain nodes v2 and compatible n8n version.  
    - *Sub-workflow:* None.

  - **OpenAI Chat Model**  
    - *Type & Role:* Represents the local OpenAI-compatible chat model interface (e.g., Ollama AI).  
    - *Configuration:* Configured to connect locally with necessary credentials or endpoint parameters for Ollama AI.  
    - *Expressions:* None apparent; serves as AI backend.  
    - *Connections:* AI language model input to "AI Agent".  
    - *Edge Cases:*  
      - Authentication or connection failures to local AI server.  
      - Model response errors or invalid outputs.  
    - *Version Specific:* Requires n8n LangChain integration supporting OpenAI chat models v1.2.

---

#### 2.4 Report Generation & Saving

- **Overview:**  
  Converts the AI output text into an HTML file and saves the report back to the local disk for review or archival.

- **Nodes Involved:**  
  - Convert to File  
  - Read/Write Files from Disk1

- **Node Details:**

  - **Convert to File**  
    - *Type & Role:* Converts text or JSON output from AI into a file format, here presumably HTML.  
    - *Configuration:* Configured to output HTML file with appropriate MIME type and filename.  
    - *Expressions:* May use expressions to dynamically set filename or content.  
    - *Connections:* Input from "AI Agent"; output to "Read/Write Files from Disk1".  
    - *Edge Cases:*  
      - Empty or malformed AI output causing empty or invalid files.  
      - File conversion errors.  
    - *Version Specific:* Requires n8n Convert to File node version 1.1 or above.

  - **Read/Write Files from Disk1**  
    - *Type & Role:* Writes the generated HTML report file to a specified location on local disk.  
    - *Configuration:* Writes file binary data to disk with configured path and filename.  
    - *Expressions:* Likely uses expressions for dynamic path setting.  
    - *Connections:* Input from "Convert to File"; no further output.  
    - *Edge Cases:*  
      - Disk permission issues or full disk errors.  
      - Overwriting existing files without backup.  
    - *Version Specific:* Standard node version 1.

---

### 3. Summary Table

| Node Name                 | Node Type                        | Functional Role                          | Input Node(s)              | Output Node(s)               | Sticky Note                             |
|---------------------------|---------------------------------|----------------------------------------|----------------------------|-----------------------------|---------------------------------------|
| Local File Trigger        | n8n-nodes-base.localFileTrigger  | Detect new/changed local files         |                            | Read/Write Files from Disk   |                                       |
| Read/Write Files from Disk | n8n-nodes-base.readWriteFile     | Read file content from disk             | Local File Trigger          | Extract from File            |                                       |
| Extract from File          | n8n-nodes-base.extractFromFile   | Extract text/content from file          | Read/Write Files from Disk  | AI Agent                    |                                       |
| AI Agent                  | @n8n/n8n-nodes-langchain.agent   | Run AI processing on extracted content | Extract from File           | Convert to File             |                                       |
| OpenAI Chat Model         | @n8n/n8n-nodes-langchain.lmChatOpenAi | Local AI language model backend      |                            | AI Agent (ai_languageModel)  |                                       |
| Convert to File           | n8n-nodes-base.convertToFile     | Convert AI output to HTML file          | AI Agent                   | Read/Write Files from Disk1  |                                       |
| Read/Write Files from Disk1| n8n-nodes-base.readWriteFile     | Write HTML report file to disk          | Convert to File            |                             |                                       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create "Local File Trigger" node:**  
   - Type: Local File Trigger  
   - Configure folder path to monitor (e.g., local folder containing legal PDFs).  
   - Set filters if needed (file types, patterns).  
   - Save.

2. **Add "Read/Write Files from Disk" node:**  
   - Type: Read/Write Files from Disk  
   - Configure it to read the file path from the trigger output.  
   - Set to read the file content (binary or text).  
   - Connect "Local File Trigger" main output to this node’s input.

3. **Add "Extract from File" node:**  
   - Type: Extract from File  
   - Configure to extract text from the incoming file content automatically.  
   - Connect output of "Read/Write Files from Disk" to this node.

4. **Add "OpenAI Chat Model" node:**  
   - Type: LangChain OpenAI Chat Model  
   - Configure credentials to point to your local Ollama AI or compatible local model endpoint.  
   - Ensure model parameters reflect your desired AI behavior and limits.

5. **Add "AI Agent" node:**  
   - Type: LangChain Agent  
   - Configure the agent to use the "OpenAI Chat Model" node as its AI language model input.  
   - Set prompts or chain logic to analyze or summarize extracted text as required.  
   - Connect "Extract from File" output to "AI Agent" main input.  
   - Connect "OpenAI Chat Model" output to "AI Agent" AI language model input.

6. **Add "Convert to File" node:**  
   - Type: Convert to File  
   - Configure to convert AI Agent output (text) into an HTML file format.  
   - Set filename (e.g., using expressions to include original file name + timestamp).  
   - Connect "AI Agent" output to "Convert to File".

7. **Add second "Read/Write Files from Disk" node (call it "Read/Write Files from Disk1"):**  
   - Type: Read/Write Files from Disk  
   - Configure to write the HTML file to a target folder on disk.  
   - Use expressions to dynamically determine path and filename as needed.  
   - Connect "Convert to File" output to this node.  
   - No further outputs needed.

8. **Ensure credentials are set up:**  
   - Local File Trigger: appropriate filesystem access.  
   - Read/Write Files nodes: permissions to read/write designated folders.  
   - OpenAI Chat Model: credentials or connection info for local Ollama AI server.

9. **Test the workflow:**  
   - Place a test legal document in the watched folder.  
   - Confirm file is detected, content extracted, processed by AI, and HTML report generated on disk.

---

### 5. General Notes & Resources

| Note Content                                                                                     | Context or Link                                                                                  |
|-------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Workflow operates 100% locally, preserving privacy for sensitive legal data.                    | Workflow Description                                                                            |
| Uses LangChain integration to connect local AI models like Ollama AI for text analysis.         | LangChain Documentation: https://js.langchain.com/docs/                                        |
| Ensure local AI model is accessible and properly configured for n8n OpenAI Chat Model node.     | Ollama AI: https://ollama.com/                                                                 |
| Local File Trigger node requires appropriate folder path and permissions on the host machine.  | n8n Docs on Local File Trigger: https://docs.n8n.io/nodes/n8n-nodes-base.localFileTrigger/      |

---

**Disclaimer:** The text provided originates exclusively from an n8n automated workflow. The process respects all content policies and contains no illegal or offensive elements. All processed data is legal and public.