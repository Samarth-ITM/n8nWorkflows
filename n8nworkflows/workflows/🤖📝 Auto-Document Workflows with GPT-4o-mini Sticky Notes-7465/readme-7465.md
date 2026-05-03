🤖📝 Auto-Document Workflows with GPT-4o-mini Sticky Notes

https://n8nworkflows.xyz/workflows/-----auto-document-workflows-with-gpt-4o-mini-sticky-notes-7465


# 🤖📝 Auto-Document Workflows with GPT-4o-mini Sticky Notes

---

## 1. Workflow Overview

This workflow automates the generation of structured documentation for n8n workflows by creating detailed sticky notes using GPT-4o-mini AI. It is designed to help users quickly draft comprehensive documentation drafts that summarize workflow components, their purposes, configurations, and interdependencies, saving time and increasing consistency.

The workflow is organized into the following logical blocks:

- **1.1 Input Reception & Context Setup**: Receives manual trigger and sets the user context describing the workflow purpose.
- **1.2 Workflow Loading and Parsing**: Loads a workflow JSON file, converts it to JSON format, and unwraps the workflow structure to identify real nodes excluding sticky notes.
- **1.3 Sticky Notes Generation per Node**: Splits the identified nodes into individual items, then generates AI-powered sticky notes describing each node’s purpose and configuration.
- **1.4 Sticky Notes Layout and Assembly**: Adjusts node and sticky note positions for readable layout (right-to-left), merges all sticky notes, assembles the final documented workflow JSON including an overview sticky note.
- **1.5 Output & Save**: Saves the fully documented workflow JSON with sticky notes to a specified file.

---

## 2. Block-by-Block Analysis

### 2.1 Input Reception & Context Setup

**Overview:**  
This block initiates the workflow manually and sets a textual description of the workflow’s purpose, which is later used as context in AI nodes for generating meaningful documentation.

**Nodes Involved:**  
- When clicking ‘Execute workflow’  
- Your Workflow Description

**Node Details:**

- **When clicking ‘Execute workflow’**  
  - Type: Manual Trigger node  
  - Role: Starts the workflow execution on user command  
  - Configuration: Default, no parameters  
  - Inputs: None  
  - Outputs: Triggers next node  
  - Edge cases: User forgets to trigger manually → workflow idle

- **Your Workflow Description**  
  - Type: Set node  
  - Role: Defines a string context variable describing the workflow purpose  
  - Configuration: Sets "Context" = "This workflow is a helper for documenting workflows. It creates a first draft of your sticky note so you don’t have to start from scratch with the documentation."  
  - Inputs: Trigger from Manual Trigger node  
  - Outputs: Provides context JSON for downstream AI nodes  
  - Edge cases: Empty or irrelevant context reduces AI output quality

---

### 2.2 Workflow Loading and Parsing

**Overview:**  
Loads an existing workflow JSON file, converts it into usable JSON data, then unwraps and filters the workflow nodes to separate real nodes from sticky notes. Produces a summary of nodes for documentation.

**Nodes Involved:**  
- load workflow  
- Convert to JSON  
- Parse Workflow

**Node Details:**

- **load workflow**  
  - Type: Read/Write File node  
  - Role: Reads workflow JSON from a specified file path  
  - Configuration: File path parameter (e.g., `/Your_path_to_file/workflow.json`)  
  - Inputs: Context from previous block  
  - Outputs: File content  
  - Edge cases: File not found, permission errors, malformed JSON

- **Convert to JSON**  
  - Type: Extract From File node  
  - Role: Parses loaded file content into JSON format  
  - Configuration: Operation: fromJson, Encoding: utf8  
  - Inputs: File content  
  - Outputs: Parsed JSON data  
  - Edge cases: Invalid JSON content causes failure

- **Parse Workflow**  
  - Type: Code node  
  - Role: Unwraps the workflow JSON to a consistent structure and filters out sticky notes  
  - Configuration: Custom JS code that handles multiple JSON structure variants and outputs:  
    - `wrappedWorkflow`: the normalized workflow object  
    - `realNodes`: array of nodes excluding sticky notes  
    - `nodesSummaryText`: JSON string summary of nodes (max 50)  
  - Inputs: Parsed JSON data  
  - Outputs: Structured workflow data for documentation  
  - Edge cases: Unexpected JSON formats trigger errors, missing nodes array fails processing

---

### 2.3 Sticky Notes Generation per Node

**Overview:**  
Splits the filtered nodes into individual items and generates AI-powered sticky notes describing each node. Also generates an overall summary sticky note documenting the entire workflow.

**Nodes Involved:**  
- Split Out Items  
- Node Sticky Notes  
- get content  
- Merge  
- Overall Sticky Note  
- Rename Overview  
- Merge1

**Node Details:**

- **Split Out Items**  
  - Type: Code node  
  - Role: Splits array of real nodes into single-item arrays for individual processing  
  - Configuration: JS code maps `realNodes` array into separate items  
  - Inputs: Filtered nodes from Parse Workflow  
  - Outputs: Individual node JSON objects  
  - Edge cases: Empty `realNodes` results in no output items

- **Node Sticky Notes**  
  - Type: OpenAI (Langchain) node  
  - Role: Generates sticky notes describing each node based on its JSON and the workflow context  
  - Configuration:  
    - Model: `gpt-4o-mini`  
    - Prompt: Custom instructions to produce clear, concise sticky notes including purpose, inputs/outputs, tips  
  - Inputs: Individual node data + workflow context  
  - Outputs: Generated sticky note content per node  
  - Credentials: Requires OpenAI API credentials  
  - Edge cases: API quota limits, malformed input JSON, network issues

- **get content**  
  - Type: Code node  
  - Role: Extracts `content` field from generated messages for further processing  
  - Configuration: JS code maps input to JSON with `content` field, defaults to empty string if missing  
  - Inputs: Output of Node Sticky Notes  
  - Outputs: Cleaned content array  
  - Edge cases: Missing content fields handled gracefully

- **Merge**  
  - Type: Merge node  
  - Role: Combines multiple input streams by position for further parallel processing  
  - Configuration: Mode: Combine by position  
  - Inputs: From `get content` and `Node Sticky Notes` outputs  
  - Outputs: Combined data stream  
  - Edge cases: Input mismatch can cause missing data in output

- **Overall Sticky Note**  
  - Type: OpenAI (Langchain) node  
  - Role: Generates a high-level documentation sticky note summarizing the entire workflow  
  - Configuration:  
    - Model: `gpt-4o-mini`  
    - Prompt includes workflow title, user context, and nodes summary text  
  - Inputs: Parsed workflow data and context  
  - Outputs: Overview sticky note content  
  - Credentials: Requires OpenAI API credentials  
  - Edge cases: Same as Node Sticky Notes (API limits, input data quality)

- **Rename Overview**  
  - Type: Code node  
  - Role: Renames the key `message.content` to `overviewContent` for consistent downstream referencing  
  - Configuration: JS extracts `message.content` and outputs as `overviewContent`  
  - Inputs: Output from Overall Sticky Note  
  - Outputs: JSON with renamed field  
  - Edge cases: Missing or malformed messages cause errors

- **Merge1**  
  - Type: Merge node  
  - Role: Combines the overview content stream and the layout blocks results  
  - Configuration: Mode: Combine by position  
  - Inputs: From `Rename Overview` and `Layout Blocks RTL`  
  - Outputs: Combined data for final assembly  
  - Edge cases: Input synchronization important to avoid missing data

---

### 2.4 Sticky Notes Layout and Assembly

**Overview:**  
Arranges nodes and their sticky notes visually from right to left for readability, applies position moves, adds an overview sticky note to the left, and assembles the final documented workflow JSON with all sticky notes included.

**Nodes Involved:**  
- Layout Blocks RTL  
- Merge2  
- Assemble Workflow

**Node Details:**

- **Layout Blocks RTL**  
  - Type: Code node  
  - Role: Calculates new positions for nodes and sticky notes to arrange them right-to-left with consistent spacing and alignment  
  - Configuration:  
    - Uses configurable constants for node width/height, sticky note dimensions, color, gaps, and row wrapping  
    - Generates new positions (`moves`) and sticky notes (`stickies`) with content per node  
  - Inputs: Combined node sticky note contents  
  - Outputs: Positioning instructions and sticky notes data  
  - Edge cases: Incorrect node position data affects layout; wrapping disabled by default (max per row large)

- **Merge2**  
  - Type: Merge node  
  - Role: Combines assembled sticky note data and parsed workflow data for final integration  
  - Configuration: Mode: Combine by position  
  - Inputs: From `Split Out Items` and `Parse Workflow` outputs  
  - Outputs: Combined data stream for assembly  
  - Edge cases: Input synchronization required

- **Assemble Workflow**  
  - Type: Code node  
  - Role: Builds the final workflow JSON including:  
    - Original workflow nodes (excluding old sticky notes)  
    - Newly generated sticky notes (overview + per node)  
    - Applies position moves to nodes  
    - Outputs base64-encoded JSON file content for saving  
  - Configuration:  
    - Uses a UUID generator for sticky note IDs  
    - Positions overview sticky note to the left of all nodes  
    - Ensures workflow root properties are set  
  - Inputs: All combined data streams including overview and layout  
  - Outputs: Final documented workflow JSON as binary data  
  - Edge cases: Missing input data throws errors; position moves must be valid

---

### 2.5 Output & Save

**Overview:**  
Writes the final documented workflow JSON file with sticky notes to a specified file path for future use or editing.

**Nodes Involved:**  
- Save documented Workflow

**Node Details:**

- **Save documented Workflow**  
  - Type: Read/Write File node  
  - Role: Saves the assembled documented workflow JSON to disk  
  - Configuration:  
    - File Name: `/Your_path_to_file/workflow-with-sticky.json`  
    - Operation: Write  
  - Inputs: Base64-encoded JSON from Assemble Workflow  
  - Outputs: None (final output)  
  - Edge cases: Permission denied errors, disk full, path invalid

---

## 3. Summary Table

| Node Name                     | Node Type                    | Functional Role                                      | Input Node(s)                   | Output Node(s)                 | Sticky Note                                                                                      |
|-------------------------------|------------------------------|-----------------------------------------------------|--------------------------------|-------------------------------|-------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger               | Manual start of workflow                             | None                           | Your Workflow Description      | Trigger the execution of the current workflow manually through the n8n interface.               |
| Your Workflow Description      | Set                          | Sets workflow context description                    | When clicking ‘Execute workflow’ | load workflow                 | Provides a brief description of the workflow's purpose for AI context.                         |
| load workflow                 | Read/Write File              | Loads workflow JSON from file                        | Your Workflow Description       | Convert to JSON                | Loads an existing workflow JSON file from specified path.                                      |
| Convert to JSON               | Extract From File            | Converts file content to JSON                        | load workflow                  | Parse Workflow                | Converts loaded data from file to JSON format for processing.                                 |
| Parse Workflow               | Code                         | Unwraps and parses workflow JSON, filters nodes     | Convert to JSON                | Overall Sticky Note, Split Out Items, Merge2 | Parses workflow JSON, extracts nodes excluding sticky notes, summarizes nodes.                 |
| Overall Sticky Note           | OpenAI (Langchain)           | Generates overall workflow documentation sticky note | Parse Workflow                | Rename Overview               | Creates a high-level sticky note summarizing the whole workflow.                              |
| Rename Overview              | Code                         | Renames output field to `overviewContent`            | Overall Sticky Note            | Merge1                       | Extracts content from AI message for consistent naming.                                       |
| Split Out Items              | Code                         | Splits array of nodes into individual node items     | Parse Workflow                | Node Sticky Notes, Merge       | Converts a list of nodes into individual items for processing.                                |
| Node Sticky Notes            | OpenAI (Langchain)           | Generates sticky notes describing each node          | Split Out Items               | get content                   | Produces detailed sticky notes for each node using AI.                                       |
| get content                  | Code                         | Extracts content field from AI-generated messages    | Node Sticky Notes             | Merge                        | Extracts `content` property safely for further processing.                                   |
| Merge                       | Merge                        | Combines content streams by position                  | get content, Node Sticky Notes | Layout Blocks RTL            | Combines multiple inputs into one stream by position.                                        |
| Layout Blocks RTL            | Code                         | Calculates node and sticky note positions (RTL)      | Merge                        | Merge1                       | Arranges sticky notes and nodes visually right-to-left with spacing and alignment.            |
| Merge1                      | Merge                        | Combines overview and layout content                  | Rename Overview, Layout Blocks RTL | Merge2                    | Merges overview sticky note with node sticky notes layout data.                              |
| Merge2                      | Merge                        | Combines parsed workflow data with sticky notes      | Parse Workflow, Split Out Items | Assemble Workflow           | Prepares all data parts for final workflow assembly.                                         |
| Assemble Workflow           | Code                         | Assembles final workflow JSON with sticky notes      | Merge2                       | Save documented Workflow      | Builds final workflow JSON, applies node moves and adds sticky notes including overview.     |
| Save documented Workflow     | Read/Write File              | Saves the documented workflow JSON file              | Assemble Workflow             | None                         | Writes the final documented workflow JSON to specified file path.                            |

---

## 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Type: Manual Trigger  
   - Name: `When clicking ‘Execute workflow’`  
   - No parameters.

2. **Create Set Node for Context**  
   - Type: Set  
   - Name: `Your Workflow Description`  
   - Add assignment:  
     - Name: `Context`  
     - Type: String  
     - Value: `"This workflow is a helper for documenting workflows. It creates a first draft of your sticky note so you don’t have to start from scratch with the documentation."`  
   - Connect from Manual Trigger.

3. **Create Read/Write File Node to Load Workflow**  
   - Type: Read/Write File  
   - Name: `load workflow`  
   - Set file selector path (e.g., `/Your_path_to_file/workflow.json`)  
   - Connect from Set node.

4. **Create Extract From File Node to Convert to JSON**  
   - Type: Extract From File  
   - Name: `Convert to JSON`  
   - Operation: `fromJson`  
   - Encoding: `utf8`  
   - Connect from `load workflow`.

5. **Create Code Node to Parse Workflow**  
   - Type: Code  
   - Name: `Parse Workflow`  
   - Paste JS code that unwraps multiple JSON formats, filters out sticky notes, and summarizes nodes (see node details)  
   - Connect from `Convert to JSON`.

6. **Create OpenAI Node for Overall Sticky Note**  
   - Type: OpenAI (Langchain)  
   - Name: `Overall Sticky Note`  
   - Model: `gpt-4o-mini`  
   - Configure prompt to generate workflow overview sticky note using workflow title, user context, and nodes summary  
   - Provide OpenAI API credentials  
   - Connect from `Parse Workflow`.

7. **Create Code Node to Rename Overview Content**  
   - Type: Code  
   - Name: `Rename Overview`  
   - JS code: Extract `message.content` as `overviewContent`  
   - Connect from `Overall Sticky Note`.

8. **Create Code Node to Split Out Items**  
   - Type: Code  
   - Name: `Split Out Items`  
   - JS code: Map `realNodes` array to individual JSON items  
   - Connect from `Parse Workflow`.

9. **Create OpenAI Node for Node Sticky Notes**  
   - Type: OpenAI (Langchain)  
   - Name: `Node Sticky Notes`  
   - Model: `gpt-4o-mini`  
   - Prompt: Generate sticky note per node JSON with detailed description and parameters  
   - Provide OpenAI API credentials  
   - Connect from `Split Out Items`.

10. **Create Code Node to Get Content**  
    - Type: Code  
    - Name: `get content`  
    - JS code: Extract `content` field safely from AI output  
    - Connect from `Node Sticky Notes`.

11. **Create Merge Node**  
    - Type: Merge  
    - Name: `Merge`  
    - Mode: Combine by position  
    - Connect inputs: from `get content` and `Node Sticky Notes`.

12. **Create Code Node for Layout Blocks RTL**  
    - Type: Code  
    - Name: `Layout Blocks RTL`  
    - Paste JS code that arranges nodes and sticky notes right-to-left, generating moves and stickies arrays  
    - Connect from `Merge`.

13. **Create Merge Node**  
    - Type: Merge  
    - Name: `Merge1`  
    - Mode: Combine by position  
    - Connect inputs: from `Rename Overview` and `Layout Blocks RTL`.

14. **Create Merge Node**  
    - Type: Merge  
    - Name: `Merge2`  
    - Mode: Combine by position  
    - Connect inputs: from `Parse Workflow` and `Split Out Items`.

15. **Create Code Node to Assemble Workflow**  
    - Type: Code  
    - Name: `Assemble Workflow`  
    - Paste JS code that assembles final workflow JSON, applies node moves, adds sticky notes including overview, outputs base64 JSON  
    - Connect from `Merge2`.

16. **Connect Merge2 Output to Assemble Workflow**  
    - Connect `Merge1` output to `Merge2` as needed to ensure all data streams combined.

17. **Create Read/Write File Node to Save Documented Workflow**  
    - Type: Read/Write File  
    - Name: `Save documented Workflow`  
    - Operation: Write  
    - File Name: `/Your_path_to_file/workflow-with-sticky.json`  
    - Connect from `Assemble Workflow`.

18. **Credentials Setup**  
    - Configure OpenAI API credentials with valid API key for nodes `Overall Sticky Note` and `Node Sticky Notes`.

19. **Final Workflow Connections**  
    - Verify all nodes connected as per the workflow connections section in the overview to ensure correct data flow.

---

## 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           | Context or Link                                     |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------|
| This workflow automates sticky note documentation generation for n8n workflows, using GPT-4o-mini AI to draft notes that help users save time and maintain consistency. It is especially useful for complex workflows requiring clear documentation.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    | Summary from Sticky Notes and workflow description |
| Contact for assistance or collaboration: Luis Acosta — Email: Luis.acosta@news2podcast.com, Twitter: [@guanchehacker](http://www.x.com/GuancheHacker)                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              | Support and collaboration contact                   |
| The workflow uses a right-to-left layout for sticky notes to improve readability and organization on the n8n editor canvas. Adjust constants in the `Layout Blocks RTL` code node to customize node and sticky note sizes, colors, and gaps.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            | Layout design guidance                              |
| Ensure OpenAI API credentials are correctly configured and have sufficient quota before running the workflow to avoid API errors.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   | Credential and API usage note                        |
| File paths for loading and saving workflow JSON files must be accessible and writable by n8n. Adjust these paths according to your environment.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   | File system access and permissions note             |
| The parser supports multiple workflow JSON formats but will throw errors if the input does not match expected structures. Validate workflow JSON files before use.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  | Workflow JSON validation                             |
| Generated sticky notes exclude existing sticky notes from the source workflow to avoid duplication; prior sticky notes are removed.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   | Sticky note management                              |

---

**Disclaimer:** The text provided is derived exclusively from an automated workflow created with n8n, a workflow automation tool. This processing strictly adheres to content policies and contains no illegal, offensive, or protected content. All processed data is legal and publicly available.