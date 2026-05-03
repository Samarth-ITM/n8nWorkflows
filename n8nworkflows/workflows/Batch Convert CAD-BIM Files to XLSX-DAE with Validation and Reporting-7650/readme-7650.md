Batch Convert CAD/BIM Files to XLSX/DAE with Validation and Reporting

https://n8nworkflows.xyz/workflows/batch-convert-cad-bim-files-to-xlsx-dae-with-validation-and-reporting-7650


# Batch Convert CAD/BIM Files to XLSX/DAE with Validation and Reporting

---

### 1. Workflow Overview

This workflow automates the batch conversion of CAD/BIM files (such as .rvt, .ifc, .dwg, .dgn) into XLSX and DAE formats, incorporating validation, reporting, and error handling. It targets architectural, engineering, and construction professionals who need automated quality control, archival, or integration of CAD data with BI tools.

**Key logical blocks:**

- **1.1 Initialization:** Trigger and load configuration parameters with pipeline start time.
- **1.2 File Discovery:** Search for CAD files matching criteria and validate presence.
- **1.3 Batch Preparation:** Split discovered files for parallel processing and prepare output directories.
- **1.4 Conversion Execution:** Convert each file using an external converter tool, capturing timing and outputs.
- **1.5 Validation:** Verify converted files’ existence and sizes, determine success or failure.
- **1.6 Reporting:** Generate a detailed HTML report with metrics, success rates, and file links.
- **1.7 Finalization:** Save the report, open it automatically, and display completion messages.

The workflow includes rich user guidance in sticky notes, configurable parameters, and robust error handling to facilitate reuse and adaptation.

---

### 2. Block-by-Block Analysis

#### 1.1 Initialization

- **Overview:** This block initiates the workflow manually or on schedule, captures the pipeline start time, and merges configuration parameters for use downstream.
- **Nodes Involved:** Manual Trigger, Capture Pipeline Start Time1, Set Configuration Parameters1, Merge Pipeline Start with Config1.
- **Node Details:**

  - **Manual Trigger**
    - Type: Trigger node to start workflow manually.
    - Config: No parameters; user manually triggers execution.
    - Outputs: Initiates pipeline start time capture.
    - Failure Modes: None expected.

  - **Capture Pipeline Start Time1**
    - Type: Code node capturing current timestamp.
    - Config: JavaScript code returns ISO timestamp and milliseconds.
    - Outputs: JSON with pipeline_start_time, pipeline_start_timestamp, and message.
    - Failure Modes: Minimal; depends on system clock.

  - **Set Configuration Parameters1**
    - Type: Set node defining core parameters.
    - Config: Editable strings and boolean:
      - converter_path: Path to RvtExporter.exe
      - source_folder: Folder to scan for CAD files
      - output_folder: Folder for converted files and reports
      - include_subfolders: boolean for recursive search
      - file_extension: type of files to convert (e.g., ".rvt")
      - options: string of flags controlling export details
    - Outputs: JSON with all configuration parameters.

  - **Merge Pipeline Start with Config1**
    - Type: Merge node combining start time and config data by position.
    - Inputs: From Capture Pipeline Start Time1 and Set Configuration Parameters1.
    - Outputs: Combined JSON to carry timing and config downstream.
    - Failure Modes: Errors if inputs are missing.

#### 1.2 File Discovery

- **Overview:** Uses PowerShell to find files by extension in source folder (optionally recursive), merges results with configuration, processes paths, and verifies files exist.
- **Nodes Involved:** Find CAD Files1, Merge Config with Search Results1, Process File List1, Check if Files Exist1.
- **Node Details:**

  - **Find CAD Files1**
    - Type: Execute Command node.
    - Config: PowerShell command to list full paths matching file extension and recursion flag.
    - Inputs: Configuration merge JSON for parameters.
    - Outputs: PowerShell stdout with file paths.
    - Failure Modes: PowerShell errors, wrong paths, permissions.

  - **Merge Config with Search Results1**
    - Type: Merge node combining configuration and search output by position.
    - Inputs: From Set Configuration Parameters1 and Find CAD Files1.
    - Outputs: Combined JSON for file processing.
    - Failure Modes: Missing inputs.

  - **Process File List1**
    - Type: Code node.
    - Config: Parses PowerShell output, normalizes paths, extracts file names, and builds file objects with expected output paths.
    - Key Variables: file_path, file_name, expected_output (.xlsx), expected_output_dae (.dae).
    - Outputs: JSON array of files with metadata, total count, and messages.
    - Failure Modes: Empty results, parsing errors, invalid paths.

  - **Check if Files Exist1**
    - Type: If node.
    - Config: Checks if files array length > 0.
    - Outputs: Branches to processing if yes; no-files response if no.
    - Failure Modes: Empty or missing file list.

#### 1.3 Batch Preparation

- **Overview:** Splits file list for individual processing, creates output directory if missing, and merges file data with configuration.
- **Nodes Involved:** Split Files for Processing1, Create Output Directory1, Merge File Data1.
- **Node Details:**

  - **Split Files for Processing1**
    - Type: SplitOut node.
    - Config: Splits array under "files" field into individual items.
    - Outputs: One item per file for parallel processing.
    - Failure Modes: Empty input arrays.

  - **Create Output Directory1**
    - Type: Execute Command node.
    - Config: PowerShell mkdir command for output folder; suppresses error if folder exists.
    - Inputs: Receives merged file/config data.
    - Outputs: Confirmation message; used for synchronization.
    - Failure Modes: Permissions, invalid path.

  - **Merge File Data1**
    - Type: Merge node.
    - Config: Combines split file data with execution context.
    - Inputs: From Split Files for Processing1 and Create Output Directory1.
    - Outputs: Full file data enriched with output directory info.
    - Failure Modes: Missing inputs.

#### 1.4 Conversion Execution

- **Overview:** For each file, capture conversion start time, delay to avoid overload, execute external conversion command, and calculate processing duration.
- **Nodes Involved:** Capture Start Time1, Small Delay, Execute Conversion1, Calculate Conversion Time1.
- **Node Details:**

  - **Capture Start Time1**
    - Type: Code node.
    - Config: Adds ISO and timestamp for conversion start; logs info.
    - Inputs: Per-file data.
    - Outputs: JSON enriched with start time fields.
    - Failure Modes: None expected.

  - **Small Delay**
    - Type: Wait node.
    - Config: Millisecond delay (default 0 ms).
    - Purpose: Avoid overloading converter or file system.
    - Failure Modes: None.

  - **Execute Conversion1**
    - Type: Execute Command node.
    - Config: Runs external "RvtExporter.exe" with arguments:
      - Input file path
      - Expected XLSX output path
      - Expected DAE output path
      - Options flags
    - Parameters templated from JSON.
    - Continue on fail enabled to allow workflow to proceed despite conversion errors.
    - Failure Modes: Command failure, invalid paths, missing executable.

  - **Calculate Conversion Time1**
    - Type: Code node.
    - Config: Calculates processing time based on conversion_start_timestamp and current time; renames stdout/stderr fields to conversion-specific.
    - Outputs: JSON with processing_time in seconds and milliseconds, conversion end time, and preserved pipeline start time.
    - Failure Modes: Missing timestamps.

#### 1.5 Validation

- **Overview:** Consolidates conversion data, verifies existence and sizes of output files, and determines success/failure per file.
- **Nodes Involved:** Merge Data Before Verification1, Verify Output Files and Get Sizes1, Complete File Verification1.
- **Node Details:**

  - **Merge Data Before Verification1**
    - Type: Merge node.
    - Config: Combines previous JSON data streams for verification.
    - Inputs: From Calculate Conversion Time1 and Capture Start Time1.
    - Outputs: Combined data for verification.
    - Failure Modes: Missing data.

  - **Verify Output Files and Get Sizes1**
    - Type: Execute Command node.
    - Config: PowerShell script checks if XLSX, DAE, and original Revit files exist; gets their file sizes.
    - Outputs: String with flags and sizes in format: `XLSX_EXISTS:True|XLSX_SIZE:12345|DAE_EXISTS:True|DAE_SIZE:67890|REVIT_EXISTS:True|REVIT_SIZE:111111`
    - Failure Modes: File access errors, path issues.

  - **Complete File Verification1**
    - Type: Code node.
    - Config: Parses verification output, extracts success booleans and sizes, determines final success criteria:
      - Both XLSX and DAE files must exist with size > 0 (unless no-collada mode)
      - Preserves all timing and config data.
    - Outputs: Detailed JSON per file with status, message, sizes, times, and pipeline info.
    - Failure Modes: Parsing errors, malformed verification data.

#### 1.6 Reporting

- **Overview:** Aggregates all verification results, calculates overall metrics, and generates a professional HTML report with styling, metrics, success/failure tables, and configuration details.
- **Nodes Involved:** Generate HTML Report1, Prepare Binary Data1, Save HTML File1.
- **Node Details:**

  - **Generate HTML Report1**
    - Type: Code node.
    - Config: Processes all verification results, calculates total files, success rate, pipeline duration, input/output sizes.
    - Handles "no-collada" mode by adjusting success logic and report notes.
    - Generates a fully styled HTML report with:
      - Metrics summary
      - Success and failure tables with file sizes and clickable links
      - Configuration snapshot
      - Timestamp and versioning info
    - Outputs: JSON with html_content, filename, output folder, and summary message.
    - Failure Modes: Large input sets, malformed data.

  - **Prepare Binary Data1**
    - Type: Code node.
    - Config: Converts HTML string to base64 binary data for file writing.
    - Outputs: Binary data property named "report".
    - Failure Modes: Missing or empty HTML content.

  - **Save HTML File1**
    - Type: Write Binary File node.
    - Config: Saves binary report file to specified path.
    - Outputs: Confirmation of file write.
    - Failure Modes: File system permissions, invalid paths.

#### 1.7 Finalization

- **Overview:** Normalizes report path for Windows, opens the HTML report automatically, and outputs final success messages.
- **Nodes Involved:** Verify and Prepare Path1, Open HTML Report1, Final Completion Notice1.
- **Node Details:**

  - **Verify and Prepare Path1**
    - Type: Code node.
    - Config: Normalizes file path for Windows, creates a CMD command string to open the file.
    - Outputs: JSON with windows_path and open command.
    - Failure Modes: Path formatting issues.

  - **Open HTML Report1**
    - Type: Execute Command node.
    - Config: Executes CMD command to open the HTML report in default browser.
    - Outputs: Confirmation.
    - Failure Modes: OS command failures.

  - **Final Completion Notice1**
    - Type: Execute Command node.
    - Config: Echoes multiple messages in console summarizing pipeline completion, report location, and next steps.
    - Failure Modes: None expected.

---

### 3. Summary Table

| Node Name                        | Node Type              | Functional Role                          | Input Node(s)                      | Output Node(s)                      | Sticky Note                                                                                                                        |
|---------------------------------|------------------------|----------------------------------------|----------------------------------|-----------------------------------|-----------------------------------------------------------------------------------------------------------------------------------|
| Manual Trigger                  | Trigger                | Start workflow manually                 | -                                | Capture Pipeline Start Time1       | Group 1: INITIALIZATION - Pipeline starts manually or scheduled, captures timestamp, loads config                                 |
| Capture Pipeline Start Time1    | Code                   | Capture pipeline start time             | Manual Trigger                   | Set Configuration Parameters1, Merge Pipeline Start with Config1 | Group 1: INITIALIZATION                                                                                                           |
| Set Configuration Parameters1  | Set                    | Define user-editable config parameters  | Capture Pipeline Start Time1    | Merge Pipeline Start with Config1 | Group 1: INITIALIZATION                                                                                                           |
| Merge Pipeline Start with Config1 | Merge                | Combine timing and config data          | Capture Pipeline Start Time1, Set Configuration Parameters1 | Find CAD Files1, Merge Config with Search Results1 | Group 1: INITIALIZATION                                                                                                           |
| Find CAD Files1                | Execute Command         | PowerShell to find matching files       | Merge Pipeline Start with Config1 | Merge Config with Search Results1 | Group 2: FILE DISCOVERY - Scans source folder with filters, lists files                                                           |
| Merge Config with Search Results1 | Merge                | Combine config and search output         | Set Configuration Parameters1, Find CAD Files1 | Process File List1               | Group 2: FILE DISCOVERY                                                                                                           |
| Process File List1             | Code                   | Parse file list, build detailed objects  | Merge Config with Search Results1 | Check if Files Exist1             | Group 2: FILE DISCOVERY                                                                                                           |
| Check if Files Exist1          | If                      | Branch based on presence of files        | Process File List1              | Split Files for Processing1, No Files Found Response1 | Group 2: FILE DISCOVERY                                                                                                           |
| No Files Found Response1       | Set                     | Set message when no files found          | Check if Files Exist1           | -                                 | -                                                                                                                                 |
| Split Files for Processing1   | SplitOut                | Split array of files into single items   | Check if Files Exist1           | Create Output Directory1, Merge File Data1 | Group 3: BATCH PREPARATION - Enables parallel processing, prepares files                                                          |
| Create Output Directory1      | Execute Command         | Ensure output directory exists            | Split Files for Processing1     | Merge File Data1                  | Group 3: BATCH PREPARATION                                                                                                        |
| Merge File Data1              | Merge                   | Combine split file data with output dir  | Split Files for Processing1, Create Output Directory1 | Capture Start Time1             | Group 3: BATCH PREPARATION                                                                                                        |
| Capture Start Time1           | Code                    | Capture conversion start time per file   | Merge File Data1                | Small Delay, Merge Data Before Verification1 | Group 4: CONVERSION EXECUTION                                                                                                     |
| Small Delay                  | Wait                    | Small delay to prevent overload           | Capture Start Time1             | Execute Conversion1               | Group 4: CONVERSION EXECUTION                                                                                                     |
| Execute Conversion1          | Execute Command         | Run external converter command            | Small Delay                    | Calculate Conversion Time1        | Group 4: CONVERSION EXECUTION                                                                                                     |
| Calculate Conversion Time1   | Code                    | Calculate conversion duration             | Execute Conversion1             | Merge Data Before Verification1   | Group 4: CONVERSION EXECUTION                                                                                                     |
| Merge Data Before Verification1 | Merge                | Combine data before verification          | Calculate Conversion Time1, Capture Start Time1 | Verify Output Files and Get Sizes1 | Group 5: VALIDATION - Consolidate data for verification                                                                           |
| Verify Output Files and Get Sizes1 | Execute Command     | Powershell check for file existence/sizes | Merge Data Before Verification1 | Complete File Verification1       | Group 5: VALIDATION                                                                                                               |
| Complete File Verification1  | Code                    | Parse verification output, decide success | Verify Output Files and Get Sizes1 | Generate HTML Report1             | Group 5: VALIDATION                                                                                                               |
| Generate HTML Report1        | Code                    | Aggregate results, generate HTML report   | Complete File Verification1     | Prepare Binary Data1              | Group 6: REPORTING - Professional report with metrics, success/failure details                                                    |
| Prepare Binary Data1         | Code                    | Convert HTML to binary for saving         | Generate HTML Report1           | Save HTML File1                  | Group 6: REPORTING                                                                                                               |
| Save HTML File1              | Write Binary File       | Save HTML report to disk                   | Prepare Binary Data1            | Verify and Prepare Path1          | Group 6: REPORTING                                                                                                               |
| Verify and Prepare Path1     | Code                    | Normalize path, prepare command to open   | Save HTML File1                | Open HTML Report1                | Group 7: FINALIZATION - Prepare report path and open automatically                                                                |
| Open HTML Report1            | Execute Command         | Open the report in default browser        | Verify and Prepare Path1        | Final Completion Notice1          | Group 7: FINALIZATION                                                                                                             |
| Final Completion Notice1     | Execute Command         | Echo success messages and instructions    | Open HTML Report1              | -                               | Group 7: FINALIZATION                                                                                                             |
| Sticky Note23                | Sticky Note             | Quick Start Guide                          | -                                | -                               | 📋 Quick Start Guide with configuration instructions                                                                             |
| Sticky Note25                | Sticky Note             | Video Tutorials                           | -                                | -                               | 🎓 Video Tutorials with useful links                                                                                              |
| Sticky Note28                | Sticky Note             | Support info                              | -                                | -                               | 🆘 Support resources and contacts                                                                                                 |
| Sticky Note29                | Sticky Note             | Use Cases                                | -                                | -                               | 🎯 Use cases for automation                                                                                                       |
| Sticky Note31                | Sticky Note             | GitHub star request                       | -                                | -                               | ⭐ Star request on GitHub                                                                                                          |
| Sticky Note                  | Sticky Note             | Options Parameter Configuration Guide    | -                                | -                               | 📋 Detailed options guide for converter flags                                                                                     |
| Sticky Note7                 | Sticky Note             | Group 1 description                       | -                                | -                               | Overview of INITIALIZATION group                                                                                                  |
| Sticky Note9                 | Sticky Note             | Group 2 description                       | -                                | -                               | Overview of FILE DISCOVERY group                                                                                                  |
| Sticky Note10                | Sticky Note             | Group 3 description                       | -                                | -                               | Overview of BATCH PREPARATION group                                                                                               |
| Sticky Note11                | Sticky Note             | Group 4 description                       | -                                | -                               | Overview of CONVERSION EXECUTION group                                                                                            |
| Sticky Note15                | Sticky Note             | Group 5 description                       | -                                | -                               | Overview of VALIDATION group                                                                                                      |
| Sticky Note16                | Sticky Note             | Group 6 description                       | -                                | -                               | Overview of REPORTING group                                                                                                       |
| Sticky Note17                | Sticky Note             | Group 7 description                       | -                                | -                               | Overview of FINALIZATION group                                                                                                    |
| Sticky Note30                | Sticky Note             | Configuration example                     | -                                | -                               | Example config snippet                                                                                                            |
| Sticky Note2                 | Sticky Note             | Instruction to only modify variables     | -                                | -                               | ⬇️ Only modify variables here                                                                                                     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger node**
   - Name: Manual Trigger
   - Purpose: Manually start the workflow.

2. **Add Code node "Capture Pipeline Start Time1"**
   - JavaScript code:
     ```js
     const now = new Date();
     return [{
       json: {
         pipeline_start_time: now.toISOString(),
         pipeline_start_timestamp: now.getTime(),
         pipeline_start_message: `Pipeline started at ${now.toLocaleString()}`
       }
     }];
     ```
   - Connect Manual Trigger → Capture Pipeline Start Time1.

3. **Create Set node "Set Configuration Parameters1"**
   - Define parameters (editable):
     - converter_path: Path to RvtExporter.exe (string)
     - source_folder: Path to CAD files folder (string)
     - output_folder: Output folder path (string)
     - include_subfolders: false (boolean)
     - file_extension: ".rvt" (string)
     - options: "basic schedule -no-collada" (string)
   - Connect Capture Pipeline Start Time1 → Set Configuration Parameters1.

4. **Add Merge node "Merge Pipeline Start with Config1"**
   - Mode: Combine by Position
   - Inputs: Capture Pipeline Start Time1 & Set Configuration Parameters1
   - Connect Set Configuration Parameters1 → Merge Pipeline Start with Config1 (index 1)
   - Connect Capture Pipeline Start Time1 → Merge Pipeline Start with Config1 (index 0)

5. **Add Execute Command node "Find CAD Files1"**
   - Command: PowerShell command to find files:
     ```powershell
     Get-ChildItem -LiteralPath '{{ $json.source_folder }}' -Filter '*{{ $json.file_extension }}' {{ $json.include_subfolders ? '-Recurse' : '' }} | Select-Object -ExpandProperty FullName
     ```
   - Connect Merge Pipeline Start with Config1 → Find CAD Files1.

6. **Add Merge node "Merge Config with Search Results1"**
   - Mode: Combine by Position
   - Inputs: Set Configuration Parameters1 & Find CAD Files1
   - Connect Find CAD Files1 → Merge Config with Search Results1 (index 1)
   - Connect Set Configuration Parameters1 → Merge Config with Search Results1 (index 0)

7. **Add Code node "Process File List1"**
   - JavaScript: Parses PowerShell output, normalizes file paths, creates metadata objects for each file including expected XLSX and DAE output paths.
   - Connect Merge Config with Search Results1 → Process File List1.

8. **Add If node "Check if Files Exist1"**
   - Condition: Check if `{{$json.files && $json.files.length || 0}} > 0`
   - True branch → Split Files for Processing1
   - False branch → No Files Found Response1

9. **Add Set node "No Files Found Response1"**
   - Set message indicating no files found, with troubleshooting hints.

10. **Add SplitOut node "Split Files for Processing1"**
    - Field to split: "files"
    - Connect Check if Files Exist1 (true) → Split Files for Processing1.

11. **Add Execute Command node "Create Output Directory1"**
    - Command: `mkdir "{{ $json.config.output_folder }}" 2>nul || echo Output directory ready`
    - Connect Split Files for Processing1 → Create Output Directory1.

12. **Add Merge node "Merge File Data1"**
    - Mode: Combine by Position
    - Inputs: Split Files for Processing1 & Create Output Directory1
    - Connect Create Output Directory1 → Merge File Data1 (index 1)
    - Connect Split Files for Processing1 → Merge File Data1 (index 0)

13. **Add Code node "Capture Start Time1"**
    - JavaScript: Adds conversion_start_time and conversion_start_timestamp to each file item.
    - Connect Merge File Data1 → Capture Start Time1.

14. **Add Wait node "Small Delay"**
    - Unit: milliseconds, default 0 (adjust for load if needed)
    - Connect Capture Start Time1 → Small Delay.

15. **Add Execute Command node "Execute Conversion1"**
    - Command:
      ```shell
      "{{ $json.converter_path }}" "{{ $json.file_path }}" "{{ $json.expected_output }}" "{{ $json.expected_output_dae }}" {{ $json.options }}
      ```
    - Continue on Fail: true
    - Connect Small Delay → Execute Conversion1.

16. **Add Code node "Calculate Conversion Time1"**
    - JS: Calculates processing time, renames stdout/stderr fields, adds conversion end time.
    - Connect Execute Conversion1 → Calculate Conversion Time1.

17. **Add Merge node "Merge Data Before Verification1"**
    - Mode: Combine by Position
    - Inputs: Calculate Conversion Time1 & Capture Start Time1
    - Connect Calculate Conversion Time1 → Merge Data Before Verification1 (index 0)
    - Connect Capture Start Time1 → Merge Data Before Verification1 (index 1)

18. **Add Execute Command node "Verify Output Files and Get Sizes1"**
    - PowerShell script checks file existence and sizes for XLSX, DAE, and original Revit files.
    - Connect Merge Data Before Verification1 → Verify Output Files and Get Sizes1.

19. **Add Code node "Complete File Verification1"**
    - Parses verification output to determine success/failure, preserves all timing/config data.
    - Connect Verify Output Files and Get Sizes1 → Complete File Verification1.

20. **Add Code node "Generate HTML Report1"**
    - Aggregates results, computes success rate, total time, input/output sizes; generates styled HTML report.
    - Connect Complete File Verification1 → Generate HTML Report1.

21. **Add Code node "Prepare Binary Data1"**
    - Converts HTML string to base64 binary for saving.
    - Connect Generate HTML Report1 → Prepare Binary Data1.

22. **Add Write Binary File node "Save HTML File1"**
    - Filename: `{{$json.full_path}}`
    - Data Property Name: "report"
    - Connect Prepare Binary Data1 → Save HTML File1.

23. **Add Code node "Verify and Prepare Path1"**
    - Normalizes report file path for Windows and prepares CMD command to open it.
    - Connect Save HTML File1 → Verify and Prepare Path1.

24. **Add Execute Command node "Open HTML Report1"**
    - Command: `cmd /c start "" "{{ $json.windows_path }}"`
    - Connect Verify and Prepare Path1 → Open HTML Report1.

25. **Add Execute Command node "Final Completion Notice1"**
    - Echoes success messages and instructions in console.
    - Connect Open HTML Report1 → Final Completion Notice1.

26. **Add necessary Sticky Notes**
    - Add all provided sticky notes for user guidance, positioning near relevant groups.

27. **Optional: Add Schedule Trigger node**
    - For scheduled automation, connect similarly as Manual Trigger.

---

### 5. General Notes & Resources

| Note Content                                                                                                       | Context or Link                                                                                                          |
|--------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------|
| 📋 Quick Start Guide: Configure parameters, execute workflow, retrieve Excel, DAE, and HTML report outputs.        | Sticky Note23                                                                                                            |
| 🎓 Video Tutorials: [CAD-BIM Pipeline Tutorial](https://www.youtube.com/watch?v=PMTZNRFjD6c), [Automated Validation](https://www.youtube.com/watch?v=p84AmP2dcvg) | Sticky Note25                                                                                                            |
| 🆘 Support: [DataDrivenConstruction.io](https://datadrivenconstruction.io), support@datadrivenconstruction.io, Community Forum: https://t.me/datadrivenconstruction | Sticky Note28                                                                                                            |
| 🎯 Use Cases: Weekly reports, quality control, archival, BI integration.                                            | Sticky Note29                                                                                                            |
| ⭐ Please star the GitHub repo to support development: [GitHub Repo](https://github.com/datadrivenconstruction/cad2data-Revit-IFC-DWG-DGN-pipeline-with-conversion-validation-qto) | Sticky Note31                                                                                                            |
| 📋 Detailed Options Parameter Guide for converter flags, export modes, feature flags, and disabling options.       | Sticky Note                                                                                                              |
| ⚠️ Configuration Example provided with paths and parameters for quick setup.                                       | Sticky Note30                                                                                                            |
| ⬇️ Only modify the variables in Set Configuration Parameters node; other workflow parts operate automatically.     | Sticky Note2                                                                                                             |

---

**Disclaimer:** The provided workflow is an automated n8n integration for legal and public data processing, compliant with content policies and free of illegal or offensive material.