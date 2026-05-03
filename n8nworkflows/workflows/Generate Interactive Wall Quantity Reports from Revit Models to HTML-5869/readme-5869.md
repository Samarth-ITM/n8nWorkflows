Generate Interactive Wall Quantity Reports from Revit Models to HTML

https://n8nworkflows.xyz/workflows/generate-interactive-wall-quantity-reports-from-revit-models-to-html-5869


# Generate Interactive Wall Quantity Reports from Revit Models to HTML

### 1. Workflow Overview

This workflow, titled **"Generate Interactive Wall Quantity Reports from Revit Models to HTML"**, automates the process of extracting quantity takeoff data from Revit BIM files, transforming it into structured summary statistics, and loading the results into a visually rich HTML report. The workflow targets construction, architecture, and BIM professionals seeking automated, reproducible quantity data extraction and reporting without manual intervention.

The workflow is logically divided into three main ETL blocks:

- **1.1 Extract Phase**: Setup and execute a Revit-to-Excel conversion tool, then read and parse the resulting Excel file into structured data.
- **1.2 Transform Phase**: Filter and clean the extracted data to focus on wall elements, aggregate quantities by wall type, and prepare summary statistics.
- **1.3 Load Phase**: Generate a polished, interactive HTML report visualizing the aggregated data and save the report file to disk.

---

### 2. Block-by-Block Analysis

#### 1.1 Extract Phase

- **Overview:**  
  This block prepares the environment for extraction, runs a Revit model converter executable to produce an Excel file with quantity data, verifies success, and reads the Excel content into n8n for processing.

- **Nodes Involved:**  
  - Start - Click to begin  
  - Setup - Define file paths  
  - Extract - Run Revit converter  
  - Check - Did extraction succeed?  
  - Success - Create Excel filename  
  - Extract - Read Excel file from disk  
  - Extract - Parse Excel to data  
  - Error - Show what went wrong  

- **Node Details:**  

  - **Start - Click to begin**  
    - Type: Manual Trigger  
    - Role: Entry point to initiate the workflow manually.  
    - Configuration: Default manual trigger with no parameters.  
    - Inputs: None  
    - Outputs: Connects to "Setup - Define file paths".  
    - Edge Cases: User must trigger manually; no automatic triggering.  

  - **Setup - Define file paths**  
    - Type: Set  
    - Role: Defines static file system paths for the Revit converter executable and the source Revit file.  
    - Configuration: Assigns two string variables:  
      - `path_to_revit_converter` pointing to the Revit converter executable location (Windows path).  
      - `revit_file` pointing to the Revit model file to process.  
    - Inputs: From Manual Trigger  
    - Outputs: To "Extract - Run Revit converter".  
    - Edge Cases: Hardcoded paths require valid file locations on the host machine; failure if paths are incorrect or inaccessible.

  - **Extract - Run Revit converter**  
    - Type: Execute Command  
    - Role: Runs an external Revit conversion executable to export the Revit model's quantity data to Excel.  
    - Configuration: Command built dynamically using the two path variables defined previously:  
      `"path_to_revit_converter" "revit_file"`  
    - Inputs: From "Setup - Define file paths"  
    - Outputs: To "Check - Did extraction succeed?"  
    - Continue On Fail: true (workflow continues even if this step fails)  
    - Edge Cases: Possible execution failure due to invalid paths, permission issues, or converter errors; output inspected for errors in next node.

  - **Check - Did extraction succeed?**  
    - Type: If  
    - Role: Checks if the extraction command reported an error.  
    - Configuration: Checks if the `error` field exists in the output JSON of the previous node.  
    - Inputs: From "Extract - Run Revit converter"  
    - Outputs:  
      - False branch (no error) → "Success - Create Excel filename"  
      - True branch (error found) → "Error - Show what went wrong"  
    - Edge Cases: If error detection logic fails or the executable produces unexpected output, failure-handling may not trigger correctly.

  - **Success - Create Excel filename**  
    - Type: Set  
    - Role: Constructs the expected Excel filename output from the Revit converter by replacing the source file extension `.rvt` with `_rvt.xlsx`.  
    - Configuration: String manipulation on the `revit_file` path.  
    - Inputs: From "Check - Did extraction succeed?" (success branch)  
    - Outputs: To "Extract - Read Excel file from disk"  
    - Edge Cases: Assumes the Excel file is named exactly as designed; mismatch leads to read errors.

  - **Extract - Read Excel file from disk**  
    - Type: Read Binary File  
    - Role: Reads the generated Excel file from disk into n8n binary data.  
    - Configuration: Reads file path from the `xlsx_filename` variable.  
    - Inputs: From "Success - Create Excel filename"  
    - Outputs: To "Extract - Parse Excel to data"  
    - Edge Cases: File not found, file locked, or permission issues cause read failure.

  - **Extract - Parse Excel to data**  
    - Type: Spreadsheet File  
    - Role: Parses Excel binary content into structured JSON rows for further processing.  
    - Configuration: Default parsing options.  
    - Inputs: From "Extract - Read Excel file from disk"  
    - Outputs: To "Transform - Filter only OST_Walls"  
    - Edge Cases: Malformed Excel files or unsupported formats may cause parse errors.

  - **Error - Show what went wrong**  
    - Type: Set  
    - Role: On extraction failure, captures and sets error message and code for downstream handling or logging.  
    - Configuration: Extracts error message and code from the converter output or defaults to generic messages.  
    - Inputs: From "Check - Did extraction succeed?" (error branch)  
    - Outputs: None (end of error path)  
    - Edge Cases: Errors in error message extraction could mask original failure details.

---

#### 1.2 Transform Phase

- **Overview:**  
  Processes raw Excel data by filtering for wall elements, cleaning and normalizing relevant fields, grouping by wall type, aggregating volume totals and counts, and preparing data for report generation.

- **Nodes Involved:**  
  - Transform - Filter only OST_Walls  
  - Transform - Clean wall data  
  - Transform - Group by Type Name & sum Volume  
  - Transform - Generate HTML Report  

- **Node Details:**  

  - **Transform - Filter only OST_Walls**  
    - Type: If  
    - Role: Filters rows where the 'Category : String' column equals 'OST_Walls', isolating wall elements only.  
    - Configuration: String equality condition on `Category : String` field.  
    - Inputs: From "Extract - Parse Excel to data"  
    - Outputs: To "Transform - Clean wall data" (true branch)  
    - Edge Cases: Rows missing 'Category : String' field or with different casing may be skipped unintentionally.

  - **Transform - Clean wall data**  
    - Type: Set  
    - Role: Normalizes and extracts required fields from filtered wall data, preparing it for aggregation.  
    - Configuration: Assigns:  
      - `type_name` from `Type Name : String` or "Unknown Type" fallback  
      - `volume` parsed as float from `Volume : Double` or zero fallback  
      - `processed_date` as ISO timestamp of current time  
    - Inputs: From "Transform - Filter only OST_Walls"  
    - Outputs: To "Transform - Group by Type Name & sum Volume"  
    - Edge Cases: Missing or malformed volume or type name fields handled with defaults.

  - **Transform - Group by Type Name & sum Volume**  
    - Type: Function  
    - Role: Groups all wall records by `type_name`, sums volumes, counts elements, calculates averages, and sorts by total volume descending.  
    - Configuration: Custom JavaScript code performing:  
      - Grouping using Array.reduce  
      - Aggregations: total volume, count, average volume (rounded to 3 decimals)  
      - Sorting results by descending total volume  
      - Adding a processing timestamp  
    - Inputs: From "Transform - Clean wall data"  
    - Outputs: To "Transform - Generate HTML Report"  
    - Edge Cases: Handles empty inputs gracefully; numeric parsing not explicitly error-trapped but uses safe defaults.

  - **Transform - Generate HTML Report**  
    - Type: Function  
    - Role: Generates an interactive, styled HTML report visualizing the grouped wall quantity data with summary cards, progress bars, and metadata.  
    - Configuration:  
      - Computes overall summary statistics (total types, total walls, total volume, average volume per wall)  
      - Builds a series of HTML cards with CSS animations and embedded styling  
      - Includes header, summary section, detailed wall type breakdown, and footer with metadata  
      - Constructs a filename based on source Revit file name appended with `_quantity_takeoff_report.html`  
      - Outputs JSON with summary and HTML content, plus binary data for saving the HTML file  
    - Inputs: From "Transform - Group by Type Name & sum Volume"  
    - Outputs: To "Load - Save HTML Report"  
    - Edge Cases: Assumes valid grouped data input; large datasets may affect browser performance on the HTML report side.

---

#### 1.3 Load Phase

- **Overview:**  
  Saves the generated HTML report as a file on disk and sets the workflow's final success message and summary outputs.

- **Nodes Involved:**  
  - Load - Save HTML Report  
  - Success - Final results  

- **Node Details:**  

  - **Load - Save HTML Report**  
    - Type: Write Binary File  
    - Role: Writes the generated HTML report binary content to disk with the specified filename.  
    - Configuration: Filename dynamically set using `filename` from previous node JSON data.  
    - Inputs: From "Transform - Generate HTML Report" (binary data)  
    - Outputs: To "Success - Final results"  
    - Edge Cases: File permission or path issues may prevent saving; no explicit error handling configured here.

  - **Success - Final results**  
    - Type: Set  
    - Role: Marks workflow completion with success flags and includes a final user message and summary object.  
    - Configuration: Sets:  
      - `success` boolean `true`  
      - `final_message` summarizing report generation and saved filename  
      - `summary` object with extracted summary statistics  
    - Inputs: From "Load - Save HTML Report"  
    - Outputs: None (end of successful workflow)  
    - Edge Cases: Assumes prior nodes succeeded; no explicit error fallback here.

---

### 3. Summary Table

| Node Name                      | Node Type           | Functional Role                                      | Input Node(s)                   | Output Node(s)                    | Sticky Note                                                                                                      |
|--------------------------------|---------------------|-----------------------------------------------------|---------------------------------|----------------------------------|-----------------------------------------------------------------------------------------------------------------|
| Start - Click to begin          | Manual Trigger      | Manual start of workflow                             | None                            | Setup - Define file paths         |                                                                                                                 |
| Setup - Define file paths       | Set                 | Defines paths for Revit converter and Revit file    | Start - Click to begin          | Extract - Run Revit converter     |                                                                                                                 |
| Extract - Run Revit converter   | Execute Command     | Runs external Revit-to-Excel converter executable   | Setup - Define file paths       | Check - Did extraction succeed?   |                                                                                                                 |
| Check - Did extraction succeed? | If                  | Detects if extraction succeeded or failed           | Extract - Run Revit converter   | Error - Show what went wrong / Success - Create Excel filename |                                                                                                                 |
| Success - Create Excel filename | Set                 | Builds Excel output filename from source file path  | Check - Did extraction succeed? | Extract - Read Excel file from disk |                                                                                                                 |
| Extract - Read Excel file from disk | Read Binary File  | Reads Excel output file from disk                    | Success - Create Excel filename | Extract - Parse Excel to data     |                                                                                                                 |
| Extract - Parse Excel to data   | Spreadsheet File    | Parses Excel binary data into structured JSON        | Extract - Read Excel file from disk | Transform - Filter only OST_Walls |                                                                                                                 |
| Transform - Filter only OST_Walls | If                | Filters rows to only include wall elements           | Extract - Parse Excel to data   | Transform - Clean wall data       |                                                                                                                 |
| Transform - Clean wall data     | Set                 | Normalizes and extracts key wall fields              | Transform - Filter only OST_Walls | Transform - Group by Type Name & sum Volume |                                                                                                                 |
| Transform - Group by Type Name & sum Volume | Function    | Groups walls by type, sums volume, counts elements   | Transform - Clean wall data     | Transform - Generate HTML Report  |                                                                                                                 |
| Transform - Generate HTML Report| Function            | Creates styled HTML report from grouped data         | Transform - Group by Type Name & sum Volume | Load - Save HTML Report      |                                                                                                                 |
| Load - Save HTML Report         | Write Binary File   | Saves HTML report file to disk                        | Transform - Generate HTML Report | Success - Final results           |                                                                                                                 |
| Success - Final results         | Set                 | Sets success flags and final message                  | Load - Save HTML Report         | None                             |                                                                                                                 |
| Extract Phase Note              | Sticky Note         | Explains Extract phase steps                          | None                           | None                             | ## 🔷 EXTRACT Phase\n\n**E**xtract data from Revit file:\n1. Setup file paths\n2. Run Revit converter (RVT → Excel)\n3. Check if conversion succeeded\n4. Read Excel file from disk\n5. Parse Excel into structured data |
| Transform Phase Note            | Sticky Note         | Explains Transform phase steps                        | None                           | None                             | ## 🔷 TRANSFORM Phase\n\n**T**ransform raw data:\n1. Filter ONLY records with 'Category : String' = 'OST_Walls'\n2. Clean data: extract ID, Type Name, Volume\n3. Group by 'Type Name : String'\n4. Sum 'Volume : Double' for each group\n5. Calculate statistics per group |
| Load Phase Note                 | Sticky Note         | Explains Load phase steps                             | None                           | None                             | ## 🔷 LOAD Phase\n\n**L**oad processed data as HTML report:\n1. Generate beautiful HTML report with embedded CSS\n2. Include summary statistics cards\n3. Display grouped data with visual progress bars\n4. Save as HTML file for easy viewing\n5. Results include:\n   - Interactive dashboard design\n   - Summary statistics\n   - Grouped data by Type Name\n   - Visual volume comparisons\n   - Professional styling |
| Sticky Note                    | Sticky Note         | General ETL and BIM data processing context          | None                           | None                             | # ETL with CAD (BIM)  \n**Extract. Transform. Load — the future of data processing in construction**\n\nETL (Extract, Transform, Load) is a time-tested and universal approach at the heart of every mature digital infrastructure. When applied to CAD and BIM data, it becomes not just relevant — but essential.\nETL is more than just a technical process. It’s a mindset shift — one that takes BIM out of the siloed world of 3D modeling and into the open world of transparent, interoperable, and machine-readable data. It is this paradigm that powers platforms like [DataDrivenConstruction.io](https://datadrivenconstruction.io) and drives the future of digital transformation in the built environment. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Name: `Start - Click to begin`  
   - Purpose: Entry point to start the workflow manually.  
   - No parameters needed.

2. **Create Set Node for File Paths**  
   - Name: `Setup - Define file paths`  
   - Assign two string variables:  
     - `path_to_revit_converter`: Full Windows path to `RvtExporter.exe` (e.g., `"C:\\DDC\\DDC_Converter\\Release\\Community\\DDC_Converter_Revit_Community\\DDC_Converter_Revit_Community\\RvtExporter.exe"`)  
     - `revit_file`: Full Windows path to source `.rvt` file (e.g., `"C:\\DDC\\DDC_Converter\\Release\\Community\\DDC_Converter_Revit_Community\\RVT_sample\\2023 racbasicsampleproject.rvt"`)  
   - Connect input from `Start - Click to begin`.

3. **Create Execute Command Node to Run Converter**  
   - Name: `Extract - Run Revit converter`  
   - Command: Expression combining both variables:  
     ```
     ="{{$json["path_to_revit_converter"]}}" "{{$json["revit_file"]}}"
     ```  
   - Set **Continue On Fail** to `true`.  
   - Connect input from `Setup - Define file paths`.

4. **Create If Node to Check Extraction Success**  
   - Name: `Check - Did extraction succeed?`  
   - Condition: Check if `error` field exists in previous node output JSON:  
     - Left Value: `{{$node["Extract - Run Revit converter"].json.error}}`  
     - Operator: Exists  
   - Connect input from `Extract - Run Revit converter`.  
   - Connect **False** branch (no error) to next success nodes, **True** branch to error handling.

5. **Create Set Node for Success Case Filename**  
   - Name: `Success - Create Excel filename`  
   - Assignment: Create `xlsx_filename` by removing `.rvt` extension from `revit_file` and appending `_rvt.xlsx`.  
   - Expression:  
     ```
     {{$node["Setup - Define file paths"].json["revit_file"].slice(0, -4) + "_rvt.xlsx"}}
     ```  
   - Connect input from `Check - Did extraction succeed?` false branch.

6. **Create Read Binary File Node**  
   - Name: `Extract - Read Excel file from disk`  
   - File Path: Use expression from `xlsx_filename` variable:  
     ```
     {{$json["xlsx_filename"]}}
     ```  
   - Connect input from `Success - Create Excel filename`.

7. **Create Spreadsheet File Node**  
   - Name: `Extract - Parse Excel to data`  
   - Use default parsing options.  
   - Connect input from `Extract - Read Excel file from disk`.

8. **Create If Node to Filter Walls Only**  
   - Name: `Transform - Filter only OST_Walls`  
   - Condition: Check if `Category : String` equals `"OST_Walls"`.  
   - Connect input from `Extract - Parse Excel to data`.

9. **Create Set Node to Clean Wall Data**  
   - Name: `Transform - Clean wall data`  
   - Assign:  
     - `type_name`: from `Type Name : String` or `"Unknown Type"` fallback  
     - `volume`: parsed float from `Volume : Double` or zero fallback  
     - `processed_date`: current date ISO string  
   - Expressions example:  
     ```
     type_name = {{$json["Type Name : String"] || "Unknown Type"}}
     volume = {{parseFloat($json["Volume : Double"]) || 0}}
     processed_date = {{new Date().toISOString()}}
     ```  
   - Connect input from `Transform - Filter only OST_Walls`.

10. **Create Function Node to Group and Aggregate Data**  
    - Name: `Transform - Group by Type Name & sum Volume`  
    - Paste JavaScript code to group walls by `type_name`, sum `volume`, count elements, calculate averages, and sort descending by total volume.  
    - Connect input from `Transform - Clean wall data`.

11. **Create Function Node to Generate HTML Report**  
    - Name: `Transform - Generate HTML Report`  
    - Paste provided JavaScript code that:  
      - Builds summary statistics  
      - Generates styled HTML with CSS and animations  
      - Creates filename for output HTML file  
      - Packages JSON summary and binary HTML data for writing  
    - Connect input from `Transform - Group by Type Name & sum Volume`.

12. **Create Write Binary File Node to Save HTML**  
    - Name: `Load - Save HTML Report`  
    - File Name: Use expression `{{$json.filename}}` from previous node.  
    - Connect input from `Transform - Generate HTML Report`.

13. **Create Set Node for Final Success Output**  
    - Name: `Success - Final results`  
    - Assign:  
      - `success`: true  
      - `final_message`: combine message and filename from previous node JSON  
      - `summary`: summary object from previous node JSON  
    - Expressions example:  
      ```
      success = true
      final_message = {{$node["Transform - Generate HTML Report"].json.message + ". File saved as: " + $node["Transform - Generate HTML Report"].json.filename}}
      summary = {{$node["Transform - Generate HTML Report"].json.summary}}
      ```  
    - Connect input from `Load - Save HTML Report`.

14. **Create Set Node for Error Reporting**  
    - Name: `Error - Show what went wrong`  
    - Assign:  
      - `error_message`: from extraction error or "Unknown error" fallback  
      - `error_code`: from extraction error code or `-1` fallback  
    - Connect input from `Check - Did extraction succeed?` true branch.

15. **Arrange Connections**  
    - Manual Trigger → Setup - Define file paths → Extract - Run Revit converter → Check - Did extraction succeed?  
    - On success branch: → Success - Create Excel filename → Extract - Read Excel file from disk → Extract - Parse Excel to data → Transform - Filter only OST_Walls → Transform - Clean wall data → Transform - Group by Type Name & sum Volume → Transform - Generate HTML Report → Load - Save HTML Report → Success - Final results  
    - On error branch: → Error - Show what went wrong  

16. **Add Sticky Notes (Optional)**  
    - Add explanatory sticky notes over each phase (Extract, Transform, Load) summarizing their steps and purpose.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                             | Context or Link                                   |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------|
| ETL (Extract, Transform, Load) is a well-established methodology for processing data, here applied to BIM data extraction and analysis from CAD models to enable automated quantity takeoff reporting. It transforms proprietary 3D model data into interoperable, actionable insights. | General workflow context                          |
| The workflow uses a community Revit converter executable (`RvtExporter.exe`) which must be installed and accessible on the host system for extraction to succeed. Paths must be updated accordingly.                                                                       | Local environment setup                            |
| The generated HTML report includes interactive CSS animations, summary cards, and visual volume progress bars to facilitate easy interpretation and presentation of wall quantities extracted from the BIM model.                                                        | Output design and usability                        |
| The workflow demonstrates an end-to-end n8n automation pipeline for BIM data extraction, emphasizing reproducibility and scalability for construction data workflows.                                                                                                  | Project and methodology overview                   |
| For more on BIM and data-driven construction, see [DataDrivenConstruction.io](https://datadrivenconstruction.io) as referenced in the sticky note.                                                                                                                      | External resource                                 |

---

**Disclaimer:**  
The provided text and workflow represent an automated process built entirely within n8n, adhering strictly to content policies. All data processed is legal, public, and free from offensive or protected content.