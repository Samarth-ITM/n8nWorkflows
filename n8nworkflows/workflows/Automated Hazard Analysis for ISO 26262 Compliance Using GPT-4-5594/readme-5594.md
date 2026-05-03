Automated Hazard Analysis for ISO 26262 Compliance Using GPT-4

https://n8nworkflows.xyz/workflows/automated-hazard-analysis-for-iso-26262-compliance-using-gpt-4-5594


# Automated Hazard Analysis for ISO 26262 Compliance Using GPT-4

### 1. Workflow Overview

This workflow automates hazard analysis for ISO 26262 compliance by leveraging GPT-4 AI to analyze system descriptions and generate a detailed hazard report. The workflow is designed for safety engineers and automotive functional safety teams aiming to identify potential hazards, root causes, and relevant ISO 26262 clauses from textual system descriptions.

The workflow is logically organized into the following blocks:

- **1.1 Input Reception:** Trigger and file reading to ingest the system description input.
- **1.2 Data Preparation:** Conversion of the read file into text format suitable for AI processing.
- **1.3 AI Hazard Analysis:** Invocation of an AI agent (based on LangChain and GPT-4) to analyze the system description and generate hazard insights.
- **1.4 Output Processing and Storage:** Conversion of AI output into a text file and saving the hazard analysis report.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
This block initiates the workflow execution manually and reads the system description file that serves as the primary input for hazard analysis.

**Nodes Involved:**  
- When clicking ‘Execute workflow’  
- Read Systems_Description  
- Convert input to binary data

**Node Details:**

- **When clicking ‘Execute workflow’**  
  - Type: Manual Trigger  
  - Role: Entry point node; triggers the workflow on user demand  
  - Configuration: Default manual trigger with no parameters  
  - Inputs: None  
  - Outputs: Connected to *Read Systems_Description*  
  - Failure Modes: None (manual trigger)  
  - Version: 1

- **Read Systems_Description**  
  - Type: Read/Write File  
  - Role: Reads system description text file from disk  
  - Configuration: Reads from `/data/inputs/1_hazard_identification/systems_description.txt`  
  - Inputs: Trigger from manual node  
  - Outputs: Binary data of the file content  
  - Failure Modes: File not found, file access permission errors  
  - Version: 1  
  - Special: `alwaysOutputData` enabled to ensure downstream nodes receive data even if file empty

- **Convert input to binary data**  
  - Type: Extract from File  
  - Role: Converts binary file data to UTF-8 text string for AI input  
  - Configuration: Operation set to `text` extraction  
  - Inputs: Binary file data from previous node  
  - Outputs: JSON with text content under `$json.data`  
  - Failure Modes: Binary to text conversion errors if file corrupt or encoding unsupported  
  - Version: 1

---

#### 1.2 AI Hazard Analysis

**Overview:**  
This block sends the system description text to a GPT-4 powered LangChain AI agent for hazard identification, requesting a structured output including potential hazards, root causes, and relevant ISO 26262 clauses.

**Nodes Involved:**  
- AI Agent  
- AI_Hazard_Analysis  
- A simple memory window

**Node Details:**

- **AI Agent**  
  - Type: LangChain Agent  
  - Role: Orchestrates AI prompt construction and sends it to the language model  
  - Configuration:  
    - Uses a custom JavaScript expression to build prompt text by reading the system description from `$json.data`  
    - Prompt requests analysis with outputs: 5 potential hazards, likely root causes, ISO 26262 relevant clauses  
    - Uses system message "You are a helpful assistant" to guide AI tone  
    - Prompt type: define (custom prompt)  
    - Output parser enabled for structured response  
  - Inputs: Text content from `Convert input to binary data`  
  - Outputs: Raw AI response JSON, connected to *Convert to File*  
  - Failure Modes: Expression errors, AI API errors, prompt formatting issues  
  - Version: 2

- **AI_Hazard_Analysis**  
  - Type: LangChain OpenAI Chat Model (GPT-4)  
  - Role: Provides the language model backend to the AI Agent  
  - Configuration:  
    - Model: `gpt-4.1-mini` (GPT-4 variant)  
    - No additional options set  
  - Inputs: AI Agent’s language model requests  
  - Credentials: Uses stored OpenAI API credentials named "OpenAi account"  
  - Outputs: AI responses back to AI Agent  
  - Failure Modes: API key invalid, rate limiting, network timeouts  
  - Version: 1.2

- **A simple memory window**  
  - Type: LangChain Memory Buffer Window  
  - Role: Preserves conversational context during AI interactions using session ID based on execution ID  
  - Configuration:  
    - Session key set dynamically to current execution ID (`{{$execution.id}}`)  
    - Session ID type: custom key  
  - Inputs: Connected to AI Agent as memory source  
  - Outputs: Feeds memory context into AI Agent for prompt enrichment  
  - Failure Modes: Memory buffer overflow, session ID misconfiguration  
  - Version: 1.3

---

#### 1.3 Output Processing and Storage

**Overview:**  
Converts the AI agent’s response to text file format and writes the hazard report to a timestamped output file for record and further use.

**Nodes Involved:**  
- Convert to File  
- Potential_risks_report.txt

**Node Details:**

- **Convert to File**  
  - Type: Convert to File  
  - Role: Converts AI agent output JSON into plain text file format  
  - Configuration:  
    - Operation: `toText`  
    - Source property: `output` (AI Agent’s output property)  
  - Inputs: Connected from AI Agent node’s main output  
  - Outputs: Text file content passed to file write node  
  - Failure Modes: Conversion errors if output property missing or malformed  
  - Version: 1.1

- **Potential_risks_report.txt**  
  - Type: Read/Write File  
  - Role: Writes the hazard analysis report to disk with timestamped filename  
  - Configuration:  
    - Operation: `write` (overwrite mode)  
    - File path: `/data/outputs/1_hazard_identification/Report_Hazard Identification.txt_{{ $now.toString() }}` uses current timestamp to avoid overwrites  
    - Append: false  
  - Inputs: Receives text file content from *Convert to File*  
  - Outputs: None (final node)  
  - Failure Modes: File write permission errors, disk full, invalid path  
  - Version: 1  
  - Special: `alwaysOutputData` enabled to confirm node completion

---

### 3. Summary Table

| Node Name                        | Node Type                              | Functional Role                             | Input Node(s)                 | Output Node(s)              | Sticky Note                                         |
|---------------------------------|--------------------------------------|---------------------------------------------|------------------------------|-----------------------------|-----------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger                       | Manual start of workflow                     | None                         | Read Systems_Description     |                                                     |
| Read Systems_Description          | Read/Write File                      | Reads system description text file          | When clicking ‘Execute workflow’ | Convert input to binary data |                                                     |
| Convert input to binary data      | Extract from File                    | Converts file binary data to UTF-8 text     | Read Systems_Description      | AI Agent                    |                                                     |
| AI Agent                        | LangChain Agent                      | Constructs prompt and manages AI interaction | Convert input to binary data   | Convert to File             |                                                     |
| AI_Hazard_Analysis               | LangChain OpenAI GPT-4 Chat          | Provides GPT-4 model for hazard analysis    | AI Agent (ai_languageModel)   | AI Agent                   |                                                     |
| A simple memory window           | LangChain Memory Buffer Window       | Maintains session context for AI prompt     | None                         | AI Agent                   |                                                     |
| Convert to File                 | Convert to File                      | Converts AI output JSON to plain text file  | AI Agent                     | Potential_risks_report.txt   |                                                     |
| Potential_risks_report.txt       | Read/Write File                      | Writes hazard report to timestamped file    | Convert to File              | None                        |                                                     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a manual trigger node**  
   - Node Type: Manual Trigger  
   - No configuration needed  
   - This node starts the workflow on user command.

2. **Add a Read/Write File node named `Read Systems_Description`**  
   - Configure to read the file from `/data/inputs/1_hazard_identification/systems_description.txt`  
   - Set operation to `read` (default)  
   - Enable `alwaysOutputData` to ensure output even if empty  
   - Connect manual trigger output to this node's input

3. **Add an Extract from File node named `Convert input to binary data`**  
   - Set operation to `text` (convert binary to UTF-8 text)  
   - Connect output of `Read Systems_Description` to this node

4. **Add a LangChain Agent node named `AI Agent`**  
   - Set `promptType` to `define` to allow custom prompt  
   - In the prompt text, use this JavaScript expression to build prompt:

   ```javascript
   const description = {{ $json.data }}.binary.data.toString('utf-8');
   const prompt = `Analyze this system for hazards:
   ${description}

   Output:
   1. 5 potential hazards
   2. Likely root causes
   3. ISO 26262 relevant clauses`;
   return [{ json: { prompt } }];
   ```

   - Set system message as `"You are a helpful assistant"`  
   - Enable `hasOutputParser`  
   - Connect output of `Convert input to binary data` to `AI Agent` input

5. **Add a LangChain OpenAI Chat Model node named `AI_Hazard_Analysis`**  
   - Choose model `gpt-4.1-mini`  
   - Connect `AI Agent` node’s `ai_languageModel` input to this node  
   - Under credentials, select or create OpenAI API credential (OAuth or API key)  
   - Use version 1.2 or above for compatibility

6. **Add a LangChain Memory Buffer Window node named `A simple memory window`**  
   - Set session key to `={{ $execution.id }}` to uniquely identify each run  
   - Set session ID type to `customKey`  
   - Connect its output to `AI Agent` node’s `ai_memory` input

7. **Connect `AI Agent` node’s main output to a Convert to File node named `Convert to File`**  
   - Set operation to `toText`  
   - Set source property as `output` (the property containing AI response text)

8. **Create a Read/Write File node named `Potential_risks_report.txt`**  
   - Set operation to `write` with append disabled  
   - Set file name to `/data/outputs/1_hazard_identification/Report_Hazard Identification.txt_{{ $now.toString() }}` to generate timestamped reports  
   - Enable `alwaysOutputData`  
   - Connect output of `Convert to File` to this node

9. **Validate all node connections:**  
   - Manual Trigger → Read Systems_Description  
   - Read Systems_Description → Convert input to binary data  
   - Convert input to binary data → AI Agent (main)  
   - AI_Hazard_Analysis → AI Agent (ai_languageModel)  
   - A simple memory window → AI Agent (ai_memory)  
   - AI Agent (main) → Convert to File  
   - Convert to File → Potential_risks_report.txt

10. **Save and activate the workflow**  
    - Test by clicking manual trigger and review output report files.

---

### 5. General Notes & Resources

| Note Content                                                                                                                     | Context or Link                                                                                 |
|----------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| This workflow relies on OpenAI GPT-4 API and LangChain integration nodes for natural language hazard analysis.                   | Requires valid OpenAI API credentials configured in n8n credentials manager.                    |
| Input system description file must be UTF-8 encoded plain text for reliable processing.                                           | File path: /data/inputs/1_hazard_identification/systems_description.txt                        |
| Outputs are saved with timestamp to prevent overwriting previous reports, facilitating audit trails.                             | Output folder: /data/outputs/1_hazard_identification/                                           |
| Ensure sufficient API quota and network connectivity to OpenAI endpoints to avoid runtime errors.                                |                                                                                               |

---

**Disclaimer:** This document is generated solely based on an automated n8n workflow. It complies strictly with content policies and contains no illegal or offensive elements. All data handled is legal and publicly accessible.