Summarize Documents, Images & Spreadsheets with Gemma 3 on Ollama

https://n8nworkflows.xyz/workflows/summarize-documents--images---spreadsheets-with-gemma-3-on-ollama-5858


# Summarize Documents, Images & Spreadsheets with Gemma 3 on Ollama

---
### 1. Workflow Overview

This workflow is designed to **automatically summarize documents, images, and spreadsheets** using the Gemma 3 AI model hosted on Ollama. It targets scenarios where files of various types (PDFs, Excel spreadsheets, CSVs, images) are placed in a local directory and need to be processed to extract and summarize their content efficiently.

The workflow consists of these logical blocks:

- **1.1 Input Reception and File Monitoring**  
  Detects new or modified files on the local disk and reads them for processing.

- **1.2 File Type Routing and Extraction**  
  Identifies the file type and routes the data through appropriate extraction nodes specialized for PDFs, XLS files, or CSVs.

- **1.3 AI Summarization Processing**  
  Sends extracted or raw file content to AI agents powered by the Ollama Chat Model (Gemma 3) to generate summaries.

- **1.4 Output Handling and Storage**  
  Converts AI responses into files and writes the summarized outputs back to disk.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and File Monitoring

- **Overview:**  
  This block monitors a local folder for any new or updated files and reads them into the workflow for further processing.

- **Nodes Involved:**  
  - Local File Trigger  
  - Read/Write Files from Disk  
  - Switch

- **Node Details:**

  - **Local File Trigger**  
    - Type: Trigger  
    - Role: Watches a configured local folder and triggers the workflow when files are added or changed.  
    - Configuration: Uses default parameters to monitor local filesystem events.  
    - Inputs: None (trigger node)  
    - Outputs: File metadata and path to the next node.  
    - Edge Cases: Permissions errors reading the folder, unsupported file types may pass through but handled later.

  - **Read/Write Files from Disk**  
    - Type: Utility  
    - Role: Reads the file content from disk based on trigger metadata.  
    - Configuration: Defaults to reading the file as binary or text depending on file type.  
    - Inputs: From Local File Trigger  
    - Outputs: File content and metadata forwarded to Switch node.  
    - Edge Cases: File read errors (missing file, locked file), large files causing timeouts.

  - **Switch**  
    - Type: Control flow  
    - Role: Routes workflow execution based on file type or other conditions derived from file metadata.  
    - Configuration: Contains multiple conditions that determine which extraction node to invoke (PDF, XLS, CSV, or default).  
    - Inputs: From Read/Write Files from Disk  
    - Outputs: One output branch per file type extraction node.  
    - Edge Cases: Unknown or unsupported file types; in such cases, the workflow may route incorrectly or fail downstream.

---

#### 2.2 File Type Routing and Extraction

- **Overview:**  
  According to the file type identified, this block extracts text or data from the file using specialized extractors.

- **Nodes Involved:**  
  - Extract from PDF  
  - Extract from XLS  
  - Extract from CSV

- **Node Details:**

  - **Extract from PDF**  
    - Type: Extractor  
    - Role: Extracts text content from PDF files for summarization.  
    - Configuration: Default PDF parsing parameters.  
    - Inputs: From Switch node (PDF branch)  
    - Outputs: Extracted text passed to AI Agent.  
    - Edge Cases: Complex PDF layouts, scanned images without OCR, encrypted PDFs.

  - **Extract from XLS**  
    - Type: Extractor  
    - Role: Extracts spreadsheet data from Excel files.  
    - Configuration: Default parsing, likely extracting cell contents.  
    - Inputs: From Switch node (XLS branch)  
    - Outputs: Extracted tabular data forwarded to AI Agent2.  
    - Edge Cases: Corrupt XLS files, password-protected spreadsheets, large sheets causing performance issues.

  - **Extract from CSV**  
    - Type: Extractor  
    - Role: Extracts data from CSV files.  
    - Configuration: Default CSV parsing parameters (delimiter, encoding).  
    - Inputs: From Switch node (CSV branch)  
    - Outputs: Parsed CSV data sent to AI Agent2.  
    - Edge Cases: Malformed CSV, encoding issues, very large files.

---

#### 2.3 AI Summarization Processing

- **Overview:**  
  This block uses AI agents configured with the Ollama Chat Model (Gemma 3) to analyze the extracted content and produce summarized text.

- **Nodes Involved:**  
  - AI Agent (3 instances named AI Agent, AI Agent1, AI Agent2)  
  - Ollama Chat Model (3 instances named Ollama Chat Model, Ollama Chat Model1, Ollama Chat Model2)

- **Node Details:**

  - **AI Agent / AI Agent1 / AI Agent2**  
    - Type: AI Agent (langchain.agent)  
    - Role: Interfaces with the Ollama Chat Model to send content for summarization and receive AI-generated summaries.  
    - Configuration: Connected to respective Ollama Chat Model nodes to specify the LLM endpoint.  
    - Inputs: Extracted content from extraction nodes or directly from Switch node (for non-extracted files).  
    - Outputs: Summarized content forwarded to the file conversion node.  
    - Edge Cases: API errors, network timeouts, malformed inputs causing generation failures.

  - **Ollama Chat Model / Ollama Chat Model1 / Ollama Chat Model2**  
    - Type: Language Model (langchain.lmChatOllama)  
    - Role: Represents the actual Ollama Gemma 3 model interface for generating natural language responses.  
    - Configuration: Uses Ollama credentials and model settings (assumed default Gemma 3).  
    - Inputs: From AI Agent nodes (ai_languageModel input)  
    - Outputs: Responses back to AI Agent nodes.  
    - Edge Cases: Authentication failures, model unavailability, rate limits.

---

#### 2.4 Output Handling and Storage

- **Overview:**  
  Converts AI-generated summaries into file formats and writes them back to disk for use or archival.

- **Nodes Involved:**  
  - Convert to File  
  - Read/Write Files from Disk1

- **Node Details:**

  - **Convert to File**  
    - Type: Utility  
    - Role: Converts the summarized text output from AI Agents into a file format suitable for saving (e.g., text or PDF).  
    - Configuration: Defaults, likely text/plain or similar format.  
    - Inputs: Summarized content from AI Agents  
    - Outputs: Binary file data forwarded to disk write node.  
    - Edge Cases: Unsupported content types, conversion errors.

  - **Read/Write Files from Disk1**  
    - Type: Utility  
    - Role: Writes the summarized file back to disk at a configured location.  
    - Configuration: Uses parameters specifying output folder and filename conventions.  
    - Inputs: From Convert to File node  
    - Outputs: Final output of workflow.  
    - Edge Cases: Write permission errors, disk full, file naming conflicts.

---

### 3. Summary Table

| Node Name               | Node Type                     | Functional Role                     | Input Node(s)              | Output Node(s)           | Sticky Note |
|-------------------------|-------------------------------|-----------------------------------|----------------------------|--------------------------|-------------|
| Sticky Note             | Sticky Note                   | Informational/Annotation          |                            |                          |             |
| Sticky Note1            | Sticky Note                   | Informational/Annotation          |                            |                          |             |
| Sticky Note2            | Sticky Note                   | Informational/Annotation          |                            |                          |             |
| Sticky Note3            | Sticky Note                   | Informational/Annotation          |                            |                          |             |
| Sticky Note4            | Sticky Note                   | Informational/Annotation          |                            |                          |             |
| Sticky Note5            | Sticky Note                   | Informational/Annotation          |                            |                          |             |
| Sticky Note6            | Sticky Note                   | Informational/Annotation          |                            |                          |             |
| Sticky Note7            | Sticky Note                   | Informational/Annotation          |                            |                          |             |
| Sticky Note8            | Sticky Note                   | Informational/Annotation          |                            |                          |             |
| Sticky Note9            | Sticky Note                   | Informational/Annotation          |                            |                          |             |
| Local File Trigger      | Trigger                      | Detect new/changed files locally  |                            | Read/Write Files from Disk |             |
| Read/Write Files from Disk | Read/Write File            | Reads files from disk              | Local File Trigger          | Switch                   |             |
| Switch                 | Switch                       | Routes by file type                | Read/Write Files from Disk  | Extract from PDF / XLS / CSV / AI Agent1 |             |
| Extract from PDF        | Extract from File             | Extract text from PDF              | Switch                     | AI Agent                 |             |
| Extract from XLS        | Extract from File             | Extract data from Excel            | Switch                     | AI Agent2                |             |
| Extract from CSV        | Extract from File             | Extract data from CSV              | Switch                     | AI Agent2                |             |
| AI Agent                | LangChain Agent               | Summarizes PDF extracted content  | Extract from PDF            | Convert to File          |             |
| AI Agent1               | LangChain Agent               | Summarizes other file content     | Switch (default route)      | Convert to File          |             |
| AI Agent2               | LangChain Agent               | Summarizes XLS and CSV content    | Extract from XLS / CSV      | Convert to File          |             |
| Ollama Chat Model       | LangChain Ollama Model        | AI model endpoint for PDF agent   | AI Agent (ai_languageModel) | AI Agent                 |             |
| Ollama Chat Model1      | LangChain Ollama Model        | AI model endpoint for AI Agent1   | AI Agent1 (ai_languageModel) | AI Agent1               |             |
| Ollama Chat Model2      | LangChain Ollama Model        | AI model endpoint for AI Agent2   | AI Agent2 (ai_languageModel) | AI Agent2               |             |
| Convert to File         | Convert To File               | Converts AI summaries to files    | AI Agent / AI Agent1 / AI Agent2 | Read/Write Files from Disk1 |             |
| Read/Write Files from Disk1 | Read/Write File            | Writes summary file to disk       | Convert to File             |                          |             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Local File Trigger node**  
   - Purpose: Monitor a local folder for new or changed files.  
   - Configure the folder path and trigger conditions as needed.

2. **Add a Read/Write Files from Disk node**  
   - Connect input from Local File Trigger.  
   - Configure to read the file content based on trigger metadata.

3. **Insert a Switch node**  
   - Connect input from Read/Write Files from Disk.  
   - Add conditions to check the file extension or MIME type to route files:  
     - PDF files → Extract from PDF node  
     - XLS files → Extract from XLS node  
     - CSV files → Extract from CSV node  
     - Others → AI Agent1 node (default route)

4. **Add Extract from PDF node**  
   - Connect input from the PDF branch of Switch.  
   - Use default settings to extract text content.

5. **Add Extract from XLS node**  
   - Connect input from the XLS branch of Switch.  
   - Use default settings to extract spreadsheet data.

6. **Add Extract from CSV node**  
   - Connect input from the CSV branch of Switch.  
   - Use default settings to parse CSV files.

7. **Create three Ollama Chat Model nodes**  
   - Ollama Chat Model (for PDF agent)  
   - Ollama Chat Model1 (for default agent)  
   - Ollama Chat Model2 (for XLS and CSV agent)  
   - Configure each with valid Ollama credentials and select the Gemma 3 model.

8. **Create three AI Agent nodes**  
   - AI Agent: Connect input from Extract from PDF and ai_languageModel input from Ollama Chat Model.  
   - AI Agent1: Connect input from Switch default branch and ai_languageModel input from Ollama Chat Model1.  
   - AI Agent2: Connect inputs from Extract from XLS and Extract from CSV; ai_languageModel input from Ollama Chat Model2.

9. **Add Convert to File node**  
   - Connect input from the main output of all AI Agent nodes.  
   - Configure to convert summarized text to a file format (e.g., text/plain).

10. **Add a Read/Write Files from Disk node (Write)**  
    - Connect input from Convert to File node.  
    - Configure output folder and filename scheme to save summaries.

11. **(Optional) Add Sticky Notes**  
    - Add sticky notes for documentation or instructions at relevant workflow sections.

---

### 5. General Notes & Resources

| Note Content                                                                                   | Context or Link                          |
|-----------------------------------------------------------------------------------------------|----------------------------------------|
| This workflow uses Gemma 3 model hosted on Ollama for advanced summarization capabilities.    | Ollama official website or docs         |
| Ensure local file system permissions are correctly configured to allow read/write operations. | n8n documentation on Local File Trigger |
| For large files or complex PDFs, consider adding error handling and timeouts for stability.  | n8n best practices for error handling  |

---

**Disclaimer:** The text provided above is exclusively derived from an n8n automated workflow and adheres strictly to content policies. No illegal or protected data is involved. All handled data is public and legal.