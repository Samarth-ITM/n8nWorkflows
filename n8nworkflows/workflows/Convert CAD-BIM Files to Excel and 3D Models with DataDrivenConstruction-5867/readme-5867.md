Convert CAD/BIM Files to Excel and 3D Models with DataDrivenConstruction

https://n8nworkflows.xyz/workflows/convert-cad-bim-files-to-excel-and-3d-models-with-datadrivenconstruction-5867


# Convert CAD/BIM Files to Excel and 3D Models with DataDrivenConstruction

### 1. Workflow Overview

This workflow automates the conversion of CAD/BIM files (specifically Revit files) into Excel spreadsheets and 3D models using the DataDrivenConstruction toolset. It targets architecture, engineering, and construction professionals who want to streamline file format conversion and data extraction from Revit project files. The workflow consists of three main logical blocks:

- **1.1 Input Initialization:** Manually triggers the workflow and sets up conversion paths.
- **1.2 Conversion Execution:** Runs the external conversion executable with specified input parameters.
- **1.3 Support & Documentation:** Provides troubleshooting guidance and community support links via sticky notes.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Initialization

- **Overview:**  
  This block starts the workflow manually and defines essential file paths for the conversion executable and the project file to be converted.

- **Nodes Involved:**  
  - When clicking ‘Test workflow’  
  - Set

- **Node Details:**

  - **When clicking ‘Test workflow’**  
    - Type: Manual Trigger  
    - Role: Entry point to start the workflow on demand.  
    - Configuration: No parameters; simply a manual activation trigger.  
    - Inputs: None  
    - Outputs: Triggers the Set node.  
    - Edge Cases: None typical, but workflow won't run if not manually triggered.  

  - **Set**  
    - Type: Set Node  
    - Role: Defines two string variables:  
      - `path_to_converter`: Absolute path to the conversion executable (`RvtExporter.exe`).  
      - `path_project_file`: Absolute path to the Revit project file (`.rvt`).  
    - Configuration: Hardcoded absolute Windows paths pointing to local files on the user’s machine.  
    - Key Variables:  
      - `path_to_converter` = `C:\Users\Artem Boiko\Desktop\n8n\cad2data-Revit-IFC-DWG-DGN-pipeline-with-conversion-validation-qto-main\cad2data-Revit-IFC-DWG-DGN-pipeline-with-conversion-validation-qto-main\DDC_Converter_Revit\datadrivenlibs\RvtExporter.exe`  
      - `path_project_file` = `C:\Users\Artem Boiko\Desktop\n8n\cad2data-Revit-IFC-DWG-DGN-pipeline-with-conversion-validation-qto-main\cad2data-Revit-IFC-DWG-DGN-pipeline-with-conversion-validation-qto-main\Sample_Projects\2023 racbasicsampleproject.rvt`  
    - Inputs: Trigger from Manual Trigger node  
    - Outputs: Triggers Execute Command node  
    - Edge Cases:  
      - If paths are incorrect or files missing, conversion will fail.  
      - Hardcoded paths reduce portability and require manual updates.  

#### 1.2 Conversion Execution

- **Overview:**  
  Executes the conversion executable with the specified project file as an argument to perform the file format transformation.

- **Nodes Involved:**  
  - Execute Command

- **Node Details:**

  - **Execute Command**  
    - Type: Execute Command (Shell command execution)  
    - Role: Runs the external Revit exporter tool with the project file as input.  
    - Configuration:  
      - Command constructed dynamically using expressions referencing `path_to_converter` and `path_project_file` variables.  
      - Command format: `"path_to_converter" "path_project_file"`  
      - `executeOnce` is expression-based, ensures command runs per execution context.  
    - Expressions:  
      - `="{{$json["path_to_converter"]}}" "{{$json["path_project_file"]}}"`  
    - Inputs: Receives paths from Set node output.  
    - Outputs: None specified (presumably outputs execution result/status).  
    - Edge Cases:  
      - Command execution failure if executable path or project file is invalid.  
      - Permissions or environment issues may cause execution errors.  
      - Long execution times or timeouts possible for large files.  
      - No explicit error handling in workflow.  

#### 1.3 Support & Documentation

- **Overview:**  
  Provides informative sticky notes within the workflow interface for troubleshooting and community engagement.

- **Nodes Involved:**  
  - Sticky Note  
  - Sticky Note1

- **Node Details:**

  - **Sticky Note**  
    - Type: Sticky Note  
    - Role: Troubleshooting guidance for executable path issues during conversion.  
    - Content:  
      - Warning that the executable must be referenced inside the `datadrivenlibs` subfolder, not at the root.  
      - Provides example path formats for clarity.  
    - Position: Near conversion nodes for visibility.  
    - Edge Cases: Helps prevent common path errors.  

  - **Sticky Note1**  
    - Type: Sticky Note  
    - Role: Community support request and acknowledgment.  
    - Content:  
      - Encourages users to star the GitHub repository to support the open-source effort.  
      - Provides GitHub link:  
        https://github.com/datadrivenconstruction/cad2data-Revit-IFC-DWG-DGN-pipeline-with-conversion-validation-qto  
    - Position: Near manual trigger node for early visibility.  

---

### 3. Summary Table

| Node Name                | Node Type          | Functional Role                           | Input Node(s)                  | Output Node(s)       | Sticky Note                                                                                               |
|--------------------------|--------------------|-----------------------------------------|-------------------------------|----------------------|----------------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger    | Starts the workflow manually             | None                          | Set                  | ⭐ **If you find our tools helpful, please consider starring our repository on [GitHub](https://github.com/datadrivenconstruction/cad2data-Revit-IFC-DWG-DGN-pipeline-with-conversion-validation-qto). Your support helps us improve and continue developing open solutions for the community!** |
| Set                      | Set                | Sets file paths for converter and project | When clicking ‘Test workflow’ | Execute Command      |                                                                                                          |
| Execute Command          | Execute Command    | Runs the conversion executable           | Set                           | None                 | ## ⚠️ Troubleshooting<br>If you encounter errors during conversion, be sure to reference the executable inside the **`datadrivenlibs`** folder. Use this path:<br><br>```text<br>"DDC_Exporter_XXXXXXX\\datadrivenlibs\\XxxExporter.exe"<br>```<br>instead of:<br>```text<br>"DDC_Exporter_XXXXXXX\\XxxExporter.exe"<br>``` |
| Sticky Note              | Sticky Note        | Troubleshooting instructions             | None                          | None                 | ## ⚠️ Troubleshooting<br>If you encounter errors during conversion, be sure to reference the executable inside the **`datadrivenlibs`** folder. Use this path:<br><br>```text<br>"DDC_Exporter_XXXXXXX\\datadrivenlibs\\XxxExporter.exe"<br>```<br>instead of:<br>```text<br>"DDC_Exporter_XXXXXXX\\XxxExporter.exe"<br>``` |
| Sticky Note1             | Sticky Note        | Support message and GitHub link          | None                          | None                 | ⭐ **If you find our tools helpful, please consider starring our repository on [GitHub](https://github.com/datadrivenconstruction/cad2data-Revit-IFC-DWG-DGN-pipeline-with-conversion-validation-qto). Your support helps us improve and continue developing open solutions for the community!** |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger node**  
   - Name: `When clicking ‘Test workflow’`  
   - Purpose: To manually start the workflow.

2. **Create a Set node**  
   - Name: `Set`  
   - Purpose: Define absolute file paths for the conversion executable and the Revit project file.  
   - Configuration: Add two string fields:  
     - `path_to_converter`: Set to your local path of `RvtExporter.exe` inside the `datadrivenlibs` folder, e.g.,  
       `C:\Users\[YourUser]\path\to\DDC_Converter_Revit\datadrivenlibs\RvtExporter.exe`  
     - `path_project_file`: Set to your local Revit project file path, e.g.,  
       `C:\Users\[YourUser]\path\to\Sample_Projects\YourProject.rvt`  
   - Connect the output of `When clicking ‘Test workflow’` to this node.

3. **Create an Execute Command node**  
   - Name: `Execute Command`  
   - Purpose: Runs the external conversion tool with the project file as argument.  
   - Configuration:  
     - Command: Use the following expression to build the command string:  
       `="{{$json["path_to_converter"]}}" "{{$json["path_project_file"]}}"`  
     - Execute Once: Set to expression mode or default so it runs once per trigger.  
   - Connect the output of the `Set` node to this node.

4. **Add two Sticky Note nodes for documentation** (optional but recommended)  
   - **Sticky Note 1:**  
     - Content:  
       ```
       ⭐ **If you find our tools helpful, please consider starring our repository on [GitHub](https://github.com/datadrivenconstruction/cad2data-Revit-IFC-DWG-DGN-pipeline-with-conversion-validation-qto). Your support helps us improve and continue developing open solutions for the community!**
       ```  
     - Position near the Manual Trigger node.

   - **Sticky Note 2:**  
     - Content:  
       ```
       ## ⚠️ Troubleshooting
       If you encounter errors during conversion, be sure to reference the executable inside the **`datadrivenlibs`** folder. Use this path:

       "DDC_Exporter_XXXXXXX\datadrivenlibs\XxxExporter.exe"

       instead of:

       "DDC_Exporter_XXXXXXX\XxxExporter.exe"
       ```  
     - Position near the Execute Command node.

5. **Verify all nodes are connected as follows:**  
   - Manual Trigger → Set → Execute Command

6. **Save and activate the workflow.**

7. **Run the workflow manually via the ‘Test workflow’ button to perform conversion.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                          | Context or Link                                                                                                                      |
|--------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------|
| ⭐ **If you find our tools helpful, please consider starring our repository on GitHub. Your support helps us improve and continue developing open solutions for the community!** | GitHub Repository: https://github.com/datadrivenconstruction/cad2data-Revit-IFC-DWG-DGN-pipeline-with-conversion-validation-qto  |
| ⚠️ When troubleshooting conversion errors, ensure the executable path points inside the `datadrivenlibs` folder as specified.       | Common error source: Using wrong executable path outside `datadrivenlibs` subfolder.                                                |

---

**Disclaimer:**  
The text provided is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and public.