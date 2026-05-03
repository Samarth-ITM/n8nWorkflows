Generate Structured Scientific Research PDF Summaries with GPT-4o

https://n8nworkflows.xyz/workflows/generate-structured-scientific-research-pdf-summaries-with-gpt-4o-5607


# Generate Structured Scientific Research PDF Summaries with GPT-4o

### 1. Workflow Overview

This n8n workflow automates the generation of structured scientific research summaries from PDF files using OpenAI's GPT-4o language model. It is designed for researchers, academics, and professionals who want to quickly obtain detailed, well-organized summaries of scientific articles without manual reading.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Detects new PDF files added to a local folder.
- **1.2 Path Normalization:** Converts Windows-style file paths to a format compatible with n8n.
- **1.3 PDF File Reading and Extraction:** Reads the detected PDF file and extracts its text content.
- **1.4 AI-Powered Summarization:** Sends the extracted text to GPT-4o with a detailed prompt instructing the model to produce a structured research summary.
- **1.5 Output Conversion and Saving:** Converts the AI output to a text file and saves it to a specified folder.
- **1.6 User Guidance and Configuration:** Provides comprehensive usage instructions and troubleshooting tips using a sticky note node.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block monitors a user-specified local folder and triggers the workflow whenever a new PDF file is added.

- **Nodes Involved:**  
  - Local File Trigger

- **Node Details:**

  - **Local File Trigger**  
    - Type: `n8n-nodes-base.localFileTrigger`  
    - Role: Watches for file system events (specifically, file additions) in a specified folder to start the workflow.  
    - Configuration:  
      - Watches for the "add" event on a folder.  
      - Folder path to monitor is set by user (not hardcoded here; configured in node UI).  
    - Input/Output: No input; outputs event data including file path of new files.  
    - Edge Cases:  
      - Permissions errors if n8n lacks access to folder.  
      - Path format issues on non-Windows systems.  
      - Large influx of files triggering multiple workflow runs in quick succession.  
    - Version: 1  

#### 2.2 Path Normalization

- **Overview:**  
  Converts Windows-style backslash file paths (`\`) into forward slashes (`/`) to ensure compatibility with downstream nodes and operating system path handling.

- **Nodes Involved:**  
  - Converter (Code Node)

- **Node Details:**

  - **Converter**  
    - Type: `n8n-nodes-base.code` (JavaScript code execution)  
    - Role: Normalizes file paths from the trigger node to a consistent format (replacing `\` with `/`).  
    - Configuration:  
      - JavaScript code that reads the path from the first input item and replaces all backslashes with forward slashes.  
      - Returns a new JSON field `convertedPath` with updated path.  
    - Input: Output from Local File Trigger.  
    - Output: Passes normalized path forward for file reading.  
    - Edge Cases:  
      - If path field is missing or malformed, code may fail.  
      - Assumes Windows-style paths; on Unix-like systems might be unnecessary but harmless.  
    - Version: 2  

#### 2.3 PDF File Reading and Extraction

- **Overview:**  
  Reads the PDF file from the normalized path and extracts its text content for AI summarization.

- **Nodes Involved:**  
  - PDF Finder (Read/Write File Node)  
  - PDF Extractor

- **Node Details:**

  - **PDF Finder**  
    - Type: `n8n-nodes-base.readWriteFile`  
    - Role: Reads the PDF file from disk using the normalized path.  
    - Configuration:  
      - Uses the `convertedPath` from the Converter node as the file selector.  
      - Reads file with default options.  
    - Input: From Converter node.  
    - Output: Passes file binary data forward.  
    - Edge Cases:  
      - File not found, permission issues, locked files.  
      - Large PDFs might cause memory issues.  
    - Version: 1  

  - **PDF Extractor**  
    - Type: `n8n-nodes-base.extractFromFile`  
    - Role: Extracts text content from the PDF binary data.  
    - Configuration:  
      - Operation set to “pdf” to extract text.  
    - Input: From PDF Finder node.  
    - Output: JSON data containing extracted text under `.text`.  
    - Edge Cases:  
      - Corrupted or scanned PDFs may yield poor or no text extraction.  
      - Large or complex PDFs might time out or partially extract.  
    - Version: 1  

#### 2.4 AI-Powered Summarization

- **Overview:**  
  Sends the extracted PDF text to OpenAI GPT-4o model with a detailed system prompt instructing it to create a structured research summary.

- **Nodes Involved:**  
  - OpenAI Chat Model  
  - Summarizer (Langchain Agent)  

- **Node Details:**

  - **OpenAI Chat Model**  
    - Type: `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
    - Role: Provides GPT-4o as the AI language model backend for Langchain agents.  
    - Configuration:  
      - Model: GPT-4o  
      - Credentials: OpenAI API key (user must create and link).  
    - Input: None directly (operates as AI provider for Summarizer).  
    - Output: Passes AI responses to connected node.  
    - Edge Cases:  
      - API key invalid, expired, or quota exceeded.  
      - Network or OpenAI server timeouts.  
    - Version: 1.2  

  - **Summarizer**  
    - Type: `@n8n/n8n-nodes-langchain.agent`  
    - Role: Acts as an AI agent to send extracted text to OpenAI GPT-4o with a custom prompt and receive structured summary.  
    - Configuration:  
      - Input text: `{{$json.text}}` (from PDF Extractor)  
      - System message: Instructs the AI to analyze the research article and provide a detailed summary following a strict layout including Introduction, Methods, Results, Summary, and Conclusion with bullet points and paragraph formatting.  
      - PromptType: "define" (fixed prompt)  
      - Connects to OpenAI Chat Model for AI calls.  
    - Input: Extracted text JSON from PDF Extractor.  
    - Output: Structured summary text under `.output`.  
    - Edge Cases:  
      - Text too long for model context size may cause truncation or errors.  
      - Model may produce incomplete or off-format summaries if prompt misunderstood.  
    - Version: 1.7  

#### 2.5 Output Conversion and Saving

- **Overview:**  
  Converts the AI-generated summary into a text file and saves it to a user-defined folder for later access.

- **Nodes Involved:**  
  - Publisher (Convert to File)  
  - Save to Folder (Read/Write File)

- **Node Details:**

  - **Publisher**  
    - Type: `n8n-nodes-base.convertToFile`  
    - Role: Converts JSON summary output into a plain text file format.  
    - Configuration:  
      - Operation: Convert to text file.  
      - Source property: `output` (the AI summary text).  
    - Input: From Summarizer.  
    - Output: File data representing the summary as a text file.  
    - Edge Cases:  
      - If summary text is empty or malformed, output file may be empty or corrupt.  
    - Version: 1.1  

  - **Save to Folder**  
    - Type: `n8n-nodes-base.readWriteFile`  
    - Role: Writes the converted text summary file to disk at user-specified path.  
    - Configuration:  
      - Operation: Write file  
      - Path to save is configured by user (should use forward slashes `/`)  
    - Input: From Publisher  
    - Output: Confirms file write success.  
    - Edge Cases:  
      - Permission issues writing to folder.  
      - Invalid or inaccessible path.  
    - Version: 1  

#### 2.6 User Guidance and Configuration

- **Overview:**  
  Provides detailed instructions for setting up the workflow, configuring API keys, adjusting folder paths, troubleshooting common issues, and suggestions for customization.

- **Nodes Involved:**  
  - Sticky Note

- **Node Details:**

  - **Sticky Note**  
    - Type: `n8n-nodes-base.stickyNote`  
    - Role: Documentation embedded inside the workflow canvas for end-user reference.  
    - Content:  
      - Step-by-step instructions for setting folder paths in the trigger and output nodes.  
      - How to create and configure OpenAI API keys with warnings about costs.  
      - Common issues such as permission errors and large PDFs exceeding token limits.  
      - Explanation of workflow operation and how to test it.  
      - Tips for customizing the AI prompt to specific scientific fields.  
    - Position: Isolated on canvas for visibility.  
    - No input/output connections.  
    - Edge Cases: N/A  

---

### 3. Summary Table

| Node Name          | Node Type                               | Functional Role                      | Input Node(s)     | Output Node(s)     | Sticky Note                                                                                                                             |
|--------------------|---------------------------------------|------------------------------------|-------------------|--------------------|-----------------------------------------------------------------------------------------------------------------------------------------|
| Local File Trigger  | n8n-nodes-base.localFileTrigger       | Detect new PDFs in folder           | -                 | Converter          | See sticky note for folder path configuration and permissions instructions.                                                             |
| Converter          | n8n-nodes-base.code                   | Normalize file path formatting      | Local File Trigger | PDF Finder         | See sticky note for path normalization explanation.                                                                                      |
| PDF Finder          | n8n-nodes-base.readWriteFile          | Read PDF file from disk             | Converter         | PDF Extractor      | See sticky note for file reading troubleshooting (run as admin if no data error).                                                        |
| PDF Extractor       | n8n-nodes-base.extractFromFile        | Extract text from PDF               | PDF Finder        | Summarizer         | See sticky note for large PDF size limitations and extraction notes.                                                                     |
| OpenAI Chat Model   | @n8n/n8n-nodes-langchain.lmChatOpenAi| Provide GPT-4o AI model             | -                 | Summarizer (AI)    | See sticky note for OpenAI API key setup and cost warnings.                                                                              |
| Summarizer          | @n8n/n8n-nodes-langchain.agent        | Analyze extracted text and summarize| PDF Extractor, OpenAI Chat Model | Publisher          | See sticky note for prompt customization and AI summarization details.                                                                   |
| Publisher           | n8n-nodes-base.convertToFile           | Convert AI output to text file      | Summarizer        | Save to Folder     | See sticky note for output format advice (txt preferred over pdf for fewer errors).                                                      |
| Save to Folder      | n8n-nodes-base.readWriteFile           | Save text summary to disk           | Publisher         | -                  | See sticky note for output path configuration and permission tips.                                                                       |
| Sticky Note         | n8n-nodes-base.stickyNote              | Provide user instructions and tips | -                 | -                  | Contains comprehensive setup, troubleshooting, cost management, and customization instructions.                                          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Local File Trigger node**  
   - Type: `Local File Trigger`  
   - Set `Trigger On` to `folder`.  
   - Configure the folder path to monitor for new PDF files (e.g., `C:/Desktop/PDF`).  
   - Set event to listen for: `add` (new files).  
   - Save the node.

2. **Add Code Node (Converter)**  
   - Type: `Code` (JavaScript).  
   - Paste the following code to replace backslashes with forward slashes:  
     ```js
     return items.map(item => {
       const originalPath = $input.first().json.path;
       const convertedPath = originalPath.replace(/\\/g, '/');
       return {
         json: {
           ...item.json,
           convertedPath,
         },
       };
     });
     ```  
   - Connect output of Local File Trigger to input of Converter.

3. **Add Read/Write File Node (PDF Finder)**  
   - Type: `Read/Write File`.  
   - Operation: Read file.  
   - Set `File Selector` to expression: `{{$json.convertedPath}}`  
   - Connect Converter output to PDF Finder input.

4. **Add Extract from File Node (PDF Extractor)**  
   - Type: `Extract From File`.  
   - Operation: PDF extraction.  
   - Connect PDF Finder output to PDF Extractor input.

5. **Add OpenAI Chat Model Node**  
   - Type: `Langchain LmChatOpenAi`.  
   - Model: Select `gpt-4o`.  
   - Create and assign OpenAI API credentials:  
     - Go to https://platform.openai.com/api-keys  
     - Create new secret key and paste into credential setup in n8n.  
   - No direct input connection required (used by agent node).

6. **Add Langchain Agent Node (Summarizer)**  
   - Type: `Langchain Agent`.  
   - Set `Text` parameter to expression: `{{$json.text}}` (output from PDF Extractor).  
   - Set `System Message` with detailed prompt:  
     - Include instructions for detailed structured summary with sections: Title, Introduction, Methods, Results, Summary, Conclusion.  
   - Set `Prompt Type` to `define`.  
   - Under credentials, select OpenAI API credentials.  
   - Connect PDF Extractor main output to Summarizer input.  
   - Connect OpenAI Chat Model AI output to Summarizer AI input.

7. **Add Convert To File Node (Publisher)**  
   - Type: `Convert To File`.  
   - Operation: Convert to text.  
   - Set `Source Property` to `output` (the AI summary text).  
   - Connect Summarizer output to Publisher input.

8. **Add Read/Write File Node (Save to Folder)**  
   - Type: `Read/Write File`.  
   - Operation: Write file.  
   - Set path parameter to desired output path (e.g., `C:/Desktop/Summary/Summary.txt`) using forward slashes `/`.  
   - Connect Publisher output to Save to Folder input.

9. **Add Sticky Note (optional but recommended)**  
   - Type: Sticky Note.  
   - Paste detailed instructions on:  
     - Folder path setup for trigger and output nodes.  
     - OpenAI API key creation and billing warnings.  
     - Troubleshooting common issues (permissions, large files).  
     - Workflow operation overview and customization tips.

10. **Set Workflow Execution Order**  
    - Ensure nodes are connected in the order listed above so data flows logically from trigger to output.

11. **Test the Workflow**  
    - Start workflow execution.  
    - Add a new PDF file to the monitored folder.  
    - Verify that the workflow triggers, processes, summarizes, and saves the summary file correctly.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               | Context or Link                                   |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------|
| Costs associated with GPT-4o usage are approximately $0.01 per run; recommended to fund your OpenAI account with no more than $5 initially to control spending.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        | Billing instructions in sticky note              |
| For better results, save summaries as `.txt` files instead of `.pdf` to avoid formatting issues.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       | Output saving advice in sticky note               |
| If "no data" errors occur reading files, run n8n as an administrator to avoid permission problems on local folders.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     | Troubleshooting section in sticky note            |
| Large PDFs may exceed OpenAI token limits causing errors; consider splitting PDFs or summarizing smaller sections if needed.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            | Limitations warning in sticky note                 |
| The system message prompt can be customized for specialized scientific fields to improve relevance, e.g., replacing "research expert" with a domain-specific expert role.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             | Customization tip in sticky note                   |
| OpenAI API key setup involves creating a secret key at https://platform.openai.com/api-keys and linking it via n8n credentials.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        | API key creation instructions in sticky note      |
| Workflow designed to automatically run when PDFs are added to the folder; manual testing requires triggering the workflow and adding files to the folder while n8n is running.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            | Operational note in sticky note                     |
| This workflow relies on Langchain integration nodes available in n8n version 1.2+; ensure n8n is updated to support these nodes.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        | Version requirement for Langchain nodes             |

---

**Disclaimer:** The text provided originates solely from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected material. All data handled is legal and publicly accessible.