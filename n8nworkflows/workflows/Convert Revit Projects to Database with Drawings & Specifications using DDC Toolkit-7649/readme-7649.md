Convert Revit Projects to Database with Drawings & Specifications using DDC Toolkit

https://n8nworkflows.xyz/workflows/convert-revit-projects-to-database-with-drawings---specifications-using-ddc-toolkit-7649


# Convert Revit Projects to Database with Drawings & Specifications using DDC Toolkit

### 1. Workflow Overview

This workflow is designed to automate the conversion of Autodesk Revit project files into structured data formats, including drawings and specifications, using the Data Driven Construction (DDC) Toolkit's Revit Converter executable. It targets construction and design professionals who need to extract and process BIM (Building Information Modeling) data into databases or other structured formats for further analysis or integration.

The workflow is logically divided into the following blocks:

- **1.1 Input Configuration and User Interaction:** Accepts user inputs either via a form UI or manual trigger, setting paths and conversion options.
- **1.2 Variable Assignment:** Sets and manages all relevant parameters for the conversion process.
- **1.3 Conversion Execution:** Runs the Revit converter executable with the configured options, handling command construction and execution.
- **1.4 Documentation and Instructions:** Provides embedded documentation, usage notes, and troubleshooting guides as sticky notes for user assistance.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Configuration and User Interaction

- **Overview:** This block manages how users provide input parameters for the conversion, either interactively through a web form or manually via a trigger. It enables flexible workflow activation methods.

- **Nodes Involved:**  
  - Form UI  
  - Manual Trigger (Optional)  
  - Sticky Note (abd24898-9349-4a9d-b141-952366d93507)  
  - Sticky Note3 (5b94ebb6-d9d7-4c2c-9be3-2b686bb7da56)

- **Node Details:**

  1. **Form UI**  
     - **Type:** Form Trigger  
     - **Role:** Provides a web form interface for users to input the path to the Revit converter executable, the Revit file path, select export mode, and conversion options.  
     - **Configuration:**  
       - Disabled by default (activation required to use).  
       - Fields include:  
         - Path to Revit Converter (string, required)  
         - Revit File Path (string, required)  
         - Export Mode (dropdown: basic, standard, complete, custom)  
         - Conversion Options (multi-select dropdown: bbox, schedule, sheets2pdf, -no-xlsx, -no-collada)  
       - Form has a descriptive title and instructions.  
     - **Connections:** No downstream connections configured (empty main output).  
     - **Edge Cases:**  
       - Popup blockers may prevent form display (noted in sticky notes).  
       - User must provide valid executable and file paths, else subsequent steps fail.  
       - Disabled by default, so must be enabled to function.

  2. **Manual Trigger (Optional)**  
     - **Type:** Manual Trigger  
     - **Role:** Allows manual activation of the workflow using pre-defined variables without user input.  
     - **Configuration:** No parameters; simply triggers downstream assignments.  
     - **Connections:** Output connected to "Set the conversion variables" node.  
     - **Edge Cases:** None specific; relies on variable correctness downstream.

  3. **Sticky Note (abd24898-9349-4a9d-b141-952366d93507)**  
     - **Purpose:** Provides instructions for using manual variables and configuring the executable path, emphasizing the executable's location inside the `datadrivenlibs` folder.  
     - **Context:** Positioned near the Manual Trigger and Set Variables nodes to guide users on configuration options.

  4. **Sticky Note3 (5b94ebb6-d9d7-4c2c-9be3-2b686bb7da56)**  
     - **Purpose:** Instructions for enabling and using the Form UI option for user input.  
     - **Context:** Positioned near the Form UI node for user guidance.

#### 2.2 Variable Assignment

- **Overview:** This block assigns all necessary variables for the conversion, including paths, export modes, and flags. It centralizes configuration for reuse in the conversion step.

- **Nodes Involved:**  
  - Set the conversion variables

- **Node Details:**

  1. **Set the conversion variables**  
     - **Type:** Set Node  
     - **Role:** Defines all parameters required for the conversion command, including paths and option flags.  
     - **Configuration:**  
       - Assigns variables with explicit paths and option strings:  
         - `path_to_revit_converter`: Absolute path to RvtExporter.exe  
         - `revit_file`: Absolute path to the Revit project file (.rvt)  
         - `(optional) Export mode`: Preset to "basic"  
         - `(optional) Add the BoundingBox geometry of each element in XLSX`: "bbox"  
         - `(optional) Export all Schedules`: "schedule"  
         - `(optional) Export all Sheets to PDF`: "sheets2pdf"  
         - `(optional) Disable export to .xlsx format`: "-no-xlsx"  
         - `(optional) Disable export to .dae format`: "-no-collada"  
         - `(optional) output_folder`: Path to output folder for generated files  
       - These options correspond to command-line flags for the converter.  
     - **Connections:** Connected downstream to the "Converting the project into a structured form1" node.  
     - **Edge Cases:**  
       - Hardcoded paths must exist and be accessible, or the conversion will fail.  
       - Optional flags might conflict or be redundant if misconfigured.

#### 2.3 Conversion Execution

- **Overview:** Executes the Revit converter executable with all configured parameters, building a command line dynamically to process the Revit file into structured outputs.

- **Nodes Involved:**  
  - Converting the project into a structured form1

- **Node Details:**

  1. **Converting the project into a structured form1**  
     - **Type:** Execute Command  
     - **Role:** Runs the external Revit Converter executable with command-line arguments assembled from the variables set previously.  
     - **Configuration:**  
       - Command is constructed using expressions that insert variables:  
         - The executable path (`path_to_revit_converter`) and Revit file path (`revit_file`).  
         - Conditional output folder and generated output filenames (`xlsx` and `dae`) are appended if output folder is defined.  
         - Export mode and optional flags concatenated to form the full command line.  
       - Executes once per trigger activation.  
     - **Connections:** No downstream nodes specified.  
     - **Edge Cases and Failure Types:**  
       - If the executable path or Revit file path is invalid, the command will fail.  
       - Output folder must be writable; otherwise, file generation fails.  
       - Command string construction may fail if variables are missing or malformed.  
       - No error handling or retries implemented in this node.  
       - Long-running execution may cause timeout based on n8n server settings.

#### 2.4 Documentation and Instructions

- **Overview:** Provides embedded instructions, troubleshooting guides, and parameter configuration details via sticky notes visible to users and maintainers.

- **Nodes Involved:**  
  - Instructions  
  - Sticky Note1  
  - Sticky Note2  
  - Sticky Note5

- **Node Details:**

  1. **Instructions**  
     - **Type:** Sticky Note  
     - **Content:** Detailed troubleshooting guide focusing on browser popup issues and correct executable path usage with examples of correct vs incorrect paths.  
     - **Context:** Positioned near the bottom left as a comprehensive help reference.

  2. **Sticky Note1**  
     - **Type:** Sticky Note  
     - **Content:** Empty (no content). Possibly reserved or placeholder node.

  3. **Sticky Note2**  
     - **Type:** Sticky Note  
     - **Content:**  
       - Detailed guide on the configuration of the Options parameter for conversion, explaining export modes, feature flags, disable options, and notes on case sensitivity and invalid combinations.  
       - Provides tables describing each mode and flag for clarity.  
     - **Context:** Positioned near the left side to accompany variable configuration.

  4. **Sticky Note5**  
     - **Type:** Sticky Note  
     - **Content:**  
       - A friendly request to star the GitHub repository of the DDC Toolkit for community support, including a direct link:  
         https://github.com/datadrivenconstruction/cad2data-Revit-IFC-DWG-DGN-pipeline-with-conversion-validation-qto  
     - **Context:** Positioned near the top right corner for visibility and promotion.

---

### 3. Summary Table

| Node Name                           | Node Type          | Functional Role                        | Input Node(s)               | Output Node(s)                   | Sticky Note                                                                                      |
|-----------------------------------|--------------------|-------------------------------------|-----------------------------|---------------------------------|-------------------------------------------------------------------------------------------------|
| Form UI                           | Form Trigger       | User input via web form              | -                           | None                            | Instructions on usage near ("Sticky Note3"): Activate to open form, fill parameters             |
| Manual Trigger (Optional)          | Manual Trigger      | Manual workflow activation           | -                           | Set the conversion variables     | Instructions on usage near ("Sticky Note"): Manual trigger with pre-set variables                |
| Set the conversion variables       | Set                 | Assign conversion paths and options  | Manual Trigger (Optional)    | Converting the project into a structured form1 | -                                                                                               |
| Converting the project into a structured form1 | Execute Command    | Runs Revit Converter executable      | Set the conversion variables | -                               | -                                                                                               |
| Instructions                      | Sticky Note         | Troubleshooting and executable path  | -                           | -                               | Troubleshooting guide for popup blockers and path issues                                        |
| Sticky Note                      | Sticky Note         | Guidance for manual variable usage   | -                           | -                               | Manual variable configuration instructions                                                     |
| Sticky Note3                     | Sticky Note         | Guidance for Form UI usage            | -                           | -                               | Instructions to activate and use Form UI                                                        |
| Sticky Note5                     | Sticky Note         | GitHub repository promotion           | -                           | -                               | Request to star GitHub repo with link: https://github.com/datadrivenconstruction/cad2data-Revit-IFC-DWG-DGN-pipeline-with-conversion-validation-qto |
| Sticky Note1                     | Sticky Note         | Empty placeholder                    | -                           | -                               | -                                                                                               |
| Sticky Note2                     | Sticky Note         | Detailed Options parameter guide     | -                           | -                               | Explanation of export modes, feature flags, disable options                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a Form Trigger node** named `Form UI`:  
   - Set `Webhook ID` to `simple-revit-converter-form`.  
   - Add form fields:  
     - "Path to Revit Converter (RvtExporter.exe)" (string, required)  
     - "Revit File Path" (string, required)  
     - "Export Mode" (dropdown) with options: basic, standard, complete, custom  
     - "Conversion Options" (multi-select dropdown) with options: bbox, schedule, sheets2pdf, -no-xlsx, -no-collada  
   - Set form title to "Revit File Converter".  
   - Keep this node disabled by default; enable if form input is desired.

3. **Add a Manual Trigger node** named `Manual Trigger (Optional)` for manual activation without user input.

4. **Add a Set node** named `Set the conversion variables`:  
   - Add string fields with the following values (these must be customized as per environment):  
     - `path_to_revit_converter`: e.g. `C:\path\to\DDC_Converter_Revit\RvtExporter.exe`  
     - `revit_file`: e.g. `C:\path\to\project.rvt`  
     - `(optional) Export mode (basic, standard, complete, custom)`: e.g. `basic`  
     - `(optional) Add the BoundingBox geometry of each element in XLSX`: e.g. `bbox`  
     - `(optional) Export all Schedules`: e.g. `schedule`  
     - `(optional) Export all Sheets to PDF`: e.g. `sheets2pdf`  
     - `(optional) Disable export to .xlsx format`: e.g. `-no-xlsx`  
     - `(optional) Disable export to .dae format`: e.g. `-no-collada`  
     - `(optional) output_folder`: e.g. `C:\output\folder\path`  
   - Connect the Manual Trigger node output to this Set node input.

5. **Add an Execute Command node** named `Converting the project into a structured form1`:  
   - Configure the command with an expression that constructs the full command line:  
     ```
     {{$json["path_to_revit_converter"]}} "{{$json["revit_file"]}}" {{ (() => {
       if ($json['(optional) output_folder'] != null) {
         const folder = $json['(optional) output_folder'];
         const file = $json['revit_file'].split("\\").slice(-1)[0];
         const xlsx = folder + "\\" + file.replace(/\.rvt$/i, '_rvt.xlsx');
         const dae  = folder + "\\" + file.replace(/\.rvt$/i, '_rvt.dae');
         return '"' + xlsx + '" ' + '"' + dae + '"';
       }
       return "";
     })() }} {{ $json['(optional) Export mode (basic, standard, complete, custom) '] }} {{ $json['(optional) Add the BoundingBox geometry of each element in XLSX '] }} {{ $json['(optional) Export all Schedules '] }} {{ $json['(optional) Export all Sheets to PDF'] }} {{ $json['(optional) Disable export to ']['xlsx format'] }} {{ $json['(optional) Disable export to ']['dae format '] }}
     ```  
   - Set "Execute Once" to true.  
   - Connect the Set node output to this Execute Command node input.

6. **Optional: Add Sticky Notes for documentation** within the workflow canvas:  
   - Troubleshooting popup blockers and executable path correctness.  
   - Instructions for manual variables and form usage.  
   - Detailed Options parameter explanation.  
   - GitHub repository link for star/support.

7. **Activate the workflow** and test using either the Manual Trigger or enabling the Form UI and submitting the form.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                    | Context or Link                                                                                                                               |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------|
| ⚠️ Troubleshooting section advises to disable browser popup blockers if form UI does not open, and to verify executable paths carefully with examples.                                                                                                                                                                                                                          | Sticky Note "Instructions"                                                                                                                    |
| Detailed parameter guide explains export modes (`basic`, `standard`, `complete`, `custom`), feature flags (`bbox`, `schedule`, `sheets2pdf`), and disabling options (`-no-xlsx`, `-no-collada`), emphasizing case sensitivity and ignoring invalid combinations.                                                                                                                     | Sticky Note2                                                                                                                                  |
| Request to star the GitHub repository to support continued development of open-source DDC tools.                                                                                                                                                                                                                                                                               | https://github.com/datadrivenconstruction/cad2data-Revit-IFC-DWG-DGN-pipeline-with-conversion-validation-qto (Sticky Note5)                   |
| The workflow is designed for Windows environments, evident from the path formatting and executable naming. Ensure paths use double backslashes (`\\`) in expressions or set accordingly in Windows style.                                                                                                                                                                      | Observed from variable paths and command construction                                                                                        |
| No explicit error handling or notifications included — users should monitor n8n execution logs for failures and ensure environment readiness (correct file paths, permissions).                                                                                                                                                                                                  | Inference based on node configurations                                                                                                       |

---

**Disclaimer:** The text provided stems exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All data handled are legal and publicly available.