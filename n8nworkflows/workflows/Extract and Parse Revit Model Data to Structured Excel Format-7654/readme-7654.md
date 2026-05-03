Extract and Parse Revit Model Data to Structured Excel Format

https://n8nworkflows.xyz/workflows/extract-and-parse-revit-model-data-to-structured-excel-format-7654


# Extract and Parse Revit Model Data to Structured Excel Format

### 1. Workflow Overview

This workflow automates the extraction and parsing of data from Autodesk Revit project files (.rvt) into a structured Excel (.xlsx) format. It targets use cases in Building Information Modeling (BIM) and Computer-Aided Design (CAD) data management, facilitating ETL (Extract, Transform, Load) processes for construction and architectural data pipelines. The workflow is designed to execute a Revit model converter tool, validate the extraction, read the resulting Excel file, and parse its content into structured data for further processing or analysis.

The workflow’s logic is grouped into the following blocks:

- **1.1 Input Reception and Setup**: Manual trigger to start the process and definition of file paths for the Revit file and the converter executable.
- **1.2 Data Extraction Execution and Validation**: Runs the external Revit converter and checks if the extraction succeeded.
- **1.3 Post-Extraction Processing**: Generates the filename for the extracted Excel file, reads it from disk, and parses its content.
- **1.4 Conditional Step (Optional)**: A placeholder conditional node that could be used for further processing based on parsed data flags.
- **1.5 Documentation and Guidance**: Several sticky notes provide context, instructions, and encouragement for users.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Setup

**Overview:**  
Starts the workflow on manual trigger and sets up necessary file path variables for the Revit converter executable and the Revit project file to process.

**Nodes Involved:**  
- Start - Click to begin  
- Setup - Define file paths

**Node Details:**  

- **Start - Click to begin**  
  - Type: Manual Trigger  
  - Role: Entry point to initiate the workflow manually.  
  - Configuration: No parameters; user triggers execution.  
  - Connections: Outputs to “Setup - Define file paths”.  
  - Edge Cases: None inherent; user must trigger manually.

- **Setup - Define file paths**  
  - Type: Set Node  
  - Role: Defines two string variables:  
    - `path_to_revit_converter`: File path to the Revit converter executable (`RvtExporter.exe`).  
    - `revit_file`: File path to the Revit project file (.rvt) to be processed.  
  - Configuration: Hardcoded absolute Windows paths assigned as strings.  
  - Key expressions: None dynamic; values are static strings.  
  - Input: Receives from Manual Trigger.  
  - Output: Passes paths to the command execution node.  
  - Edge Cases: Paths must be valid and accessible on the machine where n8n runs; otherwise, extraction will fail.

---

#### 2.2 Data Extraction Execution and Validation

**Overview:**  
Executes the external Revit converter command-line tool with the provided Revit file path, then verifies if the extraction succeeded or failed.

**Nodes Involved:**  
- Extract - Run Revit converter  
- Check - Did extraction succeed?

**Node Details:**  

- **Extract - Run Revit converter**  
  - Type: Execute Command  
  - Role: Runs the third-party Revit converter executable (`RvtExporter.exe`) with the Revit file as argument to generate Excel output.  
  - Configuration: Command constructed dynamically using expressions referencing the setup node’s file paths.  
    - Command example: `"C:\path\to\RvtExporter.exe" "C:\path\to\project.rvt"`  
  - Input: Receives paths from “Setup - Define file paths”.  
  - Output: JSON output expected to include either success data or an error field.  
  - Continue on Fail: Enabled, so downstream nodes run even if this command errors.  
  - Edge Cases:  
    - Execution failure due to invalid paths or permissions.  
    - Converter producing error output (checked downstream).  
    - Timeout or process crash.

- **Check - Did extraction succeed?**  
  - Type: If Node  
  - Role: Checks if the extraction returned an error by testing the existence of an error field in the previous node’s output JSON.  
  - Condition: Uses an expression to check if `$node["Extract - Run Revit converter"].json.error` exists.  
  - Input: From Execute Command node.  
  - Output: Two branches:  
    - False (Error exists): Leads to error handling node.  
    - True (No error): Proceeds to success path.  
  - Edge Cases: Expression failure if previous node output is malformed or missing expected fields.

---

#### 2.3 Post-Extraction Processing

**Overview:**  
On successful extraction, generates the Excel filename, reads the Excel file from disk, and parses it into structured data.

**Nodes Involved:**  
- Success - Create Excel filename  
- Extract - Read Excel file from disk  
- Extract - Parse Excel to data  
- On the standard 3D View (conditional node)

**Node Details:**  

- **Success - Create Excel filename**  
  - Type: Set Node  
  - Role: Creates a new variable `xlsx_filename` by modifying the original Revit file path, replacing the `.rvt` extension with `_rvt.xlsx`.  
  - Configuration: Uses expression:  
    - `={{ $node["Setup - Define file paths"].json["revit_file"].slice(0, -4) + "_rvt.xlsx" }}`  
  - Input: From “Check - Did extraction succeed?” (success branch).  
  - Output: Passes filename to the next node.  
  - Edge Cases: Assumes `.rvt` extension length of 4 characters; non-standard file extensions may cause incorrect naming.

- **Extract - Read Excel file from disk**  
  - Type: Read Binary File  
  - Role: Loads the generated Excel file from disk into workflow as binary data for parsing.  
  - Configuration: File path is dynamic, taken from `xlsx_filename`.  
  - Input: Receives dynamic filename from previous node.  
  - Output: Binary Excel file data for parsing.  
  - Edge Cases:  
    - File not found if extraction failed silently or naming convention mismatched.  
    - File access permission errors.

- **Extract - Parse Excel to data**  
  - Type: Spreadsheet File  
  - Role: Parses the binary Excel file into structured JSON data for downstream processing.  
  - Configuration: Default parsing options, no sheet or range filtering specified. Parses entire file.  
  - Input: Binary data from “Extract - Read Excel file from disk”.  
  - Output: JSON representation of spreadsheet data.  
  - Edge Cases: Malformed or corrupted Excel file may cause parsing errors.

- **On the standard 3D View**  
  - Type: If Node  
  - Role: Conditional branch based on a boolean property `On the standard 3D View` in parsed data. Placeholder for further conditional processing.  
  - Configuration: Checks if `$json['On the standard 3D View']` is true.  
  - Input: Parsed Excel JSON data.  
  - Output: Two branches (true/false), currently unconnected (no downstream nodes).  
  - Edge Cases: Property missing or of unexpected type may cause expression failure.

---

#### 2.4 Documentation and Guidance

**Overview:**  
Several sticky notes provide contextual documentation, instructions for users, branding, and encouragement to support the project.

**Nodes Involved:**  
- Extract Phase Note  
- Sticky Note (ETL with CAD)  
- Sticky Note1 (GitHub star encouragement)  
- Sticky Note2 (Modification instructions)  
- Extract Phase Note1 (empty content)

**Node Details:**  

- Sticky notes contain markdown-formatted text detailing:  
  - The ETL process and its importance in BIM/CAD data workflows.  
  - Step-by-step extraction phase outline.  
  - Instructions to only modify variables in the setup node.  
  - Encouragement to star the GitHub repository hosting the related tools.  
- They do not affect workflow logic but provide essential user guidance.

---

### 3. Summary Table

| Node Name                   | Node Type             | Functional Role                          | Input Node(s)                 | Output Node(s)                    | Sticky Note                                                                                         |
|-----------------------------|-----------------------|----------------------------------------|------------------------------|----------------------------------|---------------------------------------------------------------------------------------------------|
| Start - Click to begin       | Manual Trigger        | Start workflow manually                 |                              | Setup - Define file paths         |                                                                                                   |
| Setup - Define file paths    | Set                   | Define file paths variables             | Start - Click to begin         | Extract - Run Revit converter     | ## ⬇️ Only modify the variables here everything else works automatically                          |
| Extract - Run Revit converter| Execute Command       | Run external Revit converter tool       | Setup - Define file paths      | Check - Did extraction succeed?   |                                                                                                   |
| Check - Did extraction succeed?| If                   | Check if extraction succeeded           | Extract - Run Revit converter  | Error - Show what went wrong (false branch), Success - Create Excel filename (true branch) |                                                                                                   |
| Error - Show what went wrong | Set                   | Prepare error message and code          | Check - Did extraction succeed?|                               |                                                                                                   |
| Success - Create Excel filename | Set                | Generate extracted Excel filename       | Check - Did extraction succeed?| Extract - Read Excel file from disk|                                                                                                   |
| Extract - Read Excel file from disk | Read Binary File| Read generated Excel file from disk     | Success - Create Excel filename| Extract - Parse Excel to data     |                                                                                                   |
| Extract - Parse Excel to data | Spreadsheet File      | Parse Excel file to structured JSON     | Extract - Read Excel file from disk| On the standard 3D View         |                                                                                                   |
| On the standard 3D View      | If                    | Conditional check on parsed data flag   | Extract - Parse Excel to data  | (no outputs connected)            |                                                                                                   |
| Extract Phase Note           | Sticky Note           | Documentation of extraction phase       |                              |                                  | ## 🔷 EXTRACT Phase **E**xtract data from Revit file: 1. Setup file paths 2. Run converter 3. Check success 4. Read Excel 5. Parse Excel |
| Sticky Note                 | Sticky Note           | ETL and BIM data context and branding   |                              |                                  | # ETL with CAD (BIM) ETL is essential for BIM data transformation. Links to DataDrivenConstruction.io|
| Sticky Note1                | Sticky Note           | GitHub repository star encouragement    |                              |                                  | ⭐ **If you find our tools helpful**, please **consider starring** our repository on [GitHub](https://github.com/datadrivenconstruction/cad2data-Revit-IFC-DWG-DGN-pipeline-with-conversion-validation-qto). Your support helps us improve.|
| Sticky Note2                | Sticky Note           | Instruction to only modify variables     |                              |                                  | ## ⬇️ Only modify the variables here everything else works automatically                           |
| Extract Phase Note1         | Sticky Note           | Empty content placeholder                |                              |                                  |                                                                                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Name: `Start - Click to begin`  
   - Type: Manual Trigger  
   - No parameters needed.

2. **Create Set Node for File Paths**  
   - Name: `Setup - Define file paths`  
   - Type: Set  
   - Parameters:  
     - Add two string fields:  
       - `path_to_revit_converter`: Set to absolute path of `RvtExporter.exe` on your machine, e.g., `"C:\\Path\\To\\RvtExporter.exe"`  
       - `revit_file`: Set to absolute path of your Revit `.rvt` file, e.g., `"C:\\Path\\To\\Project.rvt"`  
   - Connect the Manual Trigger output to this node.

3. **Create Execute Command Node**  
   - Name: `Extract - Run Revit converter`  
   - Type: Execute Command  
   - Parameters:  
     - Command: Use expression to execute the converter with the Revit file argument:  
       `={{$json["path_to_revit_converter"]}} {{$json["revit_file"]}}`  
   - Enable “Continue On Fail” to allow workflow continuation on errors.  
   - Connect “Setup - Define file paths” output to this node.

4. **Create If Node to Check Extraction Success**  
   - Name: `Check - Did extraction succeed?`  
   - Type: If  
   - Condition: Check if error property exists in previous node output:  
     - Expression: `{{$node["Extract - Run Revit converter"].json.error}}` exists  
   - Connect “Extract - Run Revit converter” output to this node.

5. **Create Set Node for Error Handling**  
   - Name: `Error - Show what went wrong`  
   - Type: Set  
   - Parameters:  
     - `error_message`: Expression: `=Extraction failed: {{$node["Extract - Run Revit converter"].json.error || "Unknown error"}}`  
     - `error_code`: Expression: `={{$node["Extract - Run Revit converter"].json.code || -1}}`  
   - Connect the False output (error path) of the If node to this node.

6. **Create Set Node to Generate Excel Filename**  
   - Name: `Success - Create Excel filename`  
   - Type: Set  
   - Parameters:  
     - `xlsx_filename`: Expression:  
       `={{ $node["Setup - Define file paths"].json["revit_file"].slice(0, -4) + "_rvt.xlsx" }}`  
   - Connect the True output (success path) of the If node to this node.

7. **Create Read Binary File Node**  
   - Name: `Extract - Read Excel file from disk`  
   - Type: Read Binary File  
   - Parameters:  
     - File Path: Set to `={{ $json["xlsx_filename"] }}` (dynamic from previous node)  
   - Connect “Success - Create Excel filename” output to this node.

8. **Create Spreadsheet File Node to Parse Excel**  
   - Name: `Extract - Parse Excel to data`  
   - Type: Spreadsheet File  
   - Parameters: Default parsing options (parse entire file)  
   - Connect “Extract - Read Excel file from disk” output to this node.

9. **Create Optional If Node for Further Processing**  
   - Name: `On the standard 3D View`  
   - Type: If  
   - Condition: Check if boolean field `On the standard 3D View` is true in parsed JSON data:  
     - Expression: `={{ $json['On the standard 3D View'] === true }}`  
   - Connect “Extract - Parse Excel to data” output to this node.

10. **Create Sticky Notes as Documentation** (Optional)  
    - Add sticky notes with markdown content to explain workflow purpose, instructions, and branding references.

---

### 5. General Notes & Resources

| Note Content                                                                                                                    | Context or Link                                                                                              |
|--------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------|
| ETL (Extract, Transform, Load) is critical for bringing BIM data into interoperable, machine-readable formats.                 | Conceptual explanation of workflow context.                                                                 |
| This workflow uses an external Revit converter executable (`RvtExporter.exe`) to extract data from `.rvt` files to Excel.      | Requires local installation of the converter tool accessible by n8n.                                        |
| GitHub repository for the related tools and pipelines: https://github.com/datadrivenconstruction/cad2data-Revit-IFC-DWG-DGN-pipeline-with-conversion-validation-qto | Project repository for source code, issues, and community support.                                           |
| Users should only modify the file path variables in the setup node; all other workflow logic is automated.                     | Simplifies maintenance and reduces risk of workflow breakage.                                               |
| The workflow assumes Windows path conventions and `.rvt` extension length for filename manipulations.                           | May require adjustments for other OS or file naming conventions.                                           |
| The “On the standard 3D View” conditional node is a placeholder for downstream logic depending on parsed data flags.           | Customizable depending on project requirements.                                                             |

---

**Disclaimer:** The provided text is exclusively derived from an automated n8n workflow. It complies with all current content policies and contains no illegal or protected material. All handled data is legal and publicly accessible.