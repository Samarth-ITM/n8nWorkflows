Estimate Construction Costs from Revit/IFC Models with GPT-4 and Claude

https://n8nworkflows.xyz/workflows/estimate-construction-costs-from-revit-ifc-models-with-gpt-4-and-claude-7652


# Estimate Construction Costs from Revit/IFC Models with GPT-4 and Claude

---
### 1. Workflow Overview

This workflow automates the estimation of construction costs from BIM data extracted via Revit or IFC models, leveraging advanced Large Language Models (LLMs) like GPT-4 and Claude (Anthropic) for classification, material analysis, and pricing insights.

**Target Use Cases:**  
- Construction cost estimation from architectural BIM data (Revit/IFC)  
- Automated classification of building elements vs. non-building elements  
- Material classification according to multiple standards (EU, German, US)  
- Price estimation combining AI-driven analysis and online data sources  
- Generation of professional analytic reports (HTML + Excel) for project stakeholders

**Logical Blocks:**

- **Block 1. Data Loading & Grouping:**  
  Loading and parsing Excel data extracted from Revit/IFC files, cleaning headers, AI-based aggregation rule determination, and grouping data by a chosen parameter.

- **Block 2. Element Classification:**  
  Identification of category fields, AI classification of elements into building vs. non-building elements, and flow splitting accordingly.

- **Block 3. Material Analysis & Price Estimation:**  
  Batch processing of building elements, detailed AI analysis for material classification and price estimation, result accumulation.

- **Block 4. Price Calculation & Reporting:**  
  Aggregation of all results, calculation of project totals, generating professional HTML reports with charts, and creating multi-sheet Excel exports.

- **Conversion Block:**  
  Checks for existing Excel extraction; if missing, runs an external converter to produce the Excel file from the Revit project.

---

### 2. Block-by-Block Analysis

---

#### Block 1. Data Loading & Grouping

**Overview:**  
This block loads the extracted Excel file, cleans and maps headers, uses AI to assign aggregation rules (sum, mean, or first) for each column, and groups raw data by a specified parameter.

**Nodes Involved:**  
- Setup - Define file paths  
- Create - Excel filename1  
- Check - Does Excel file exist?1  
- If - File exists?1  
- Extract - Run converter1  
- Check - Did extraction succeed?1  
- Error - Show what went wrong1  
- Set xlsx_filename after success1  
- Merge - Continue workflow1  
- Set Parameters1  
- Read Excel File1  
- Parse Excel1  
- Extract Headers and Data  
- AI Analyze All Headers  
- Process AI Response1  
- Group Data with AI Rules1  
- On the standard 3D View  
- Find Category Fields1  

**Node Details:**

- **Setup - Define file paths**  
  *Type:* Set node  
  *Role:* Defines critical paths and parameters: path to Revit converter executable, Revit project file path, grouping parameter (e.g., "Type Name"), and target country for price localization.  
  *Inputs/Outputs:* Entry point; outputs parameters for downstream nodes.  
  *Potential failures:* Incorrect paths or missing files cause downstream errors.

- **Create - Excel filename1**  
  *Type:* Set node  
  *Role:* Constructs the expected Excel filename derived from the Revit project filename.  
  *Inputs:* Receives project file path.  
  *Outputs:* Supplies Excel filename for existence checks.

- **Check - Does Excel file exist?1**  
  *Type:* Read Binary File node  
  *Role:* Checks if the Excel file already exists to skip unnecessary conversion.  
  *Error handling:* Continues on fail to allow conversion if file missing.

- **If - File exists?1**  
  *Type:* If node  
  *Role:* Routes flow based on whether Excel file exists or not.

- **Extract - Run converter1**  
  *Type:* Execute Command  
  *Role:* Runs external Revit to Excel converter executable if Excel file is absent.  
  *Parameters:* Uses paths set upstream.

- **Check - Did extraction succeed?1**  
  *Type:* If node  
  *Role:* Verifies if converter ran successfully by checking error fields.  
  *Failure path:* Routes to an error display node.

- **Error - Show what went wrong1**  
  *Type:* Set node  
  *Role:* Captures and formats extraction failure messages for logging.

- **Set xlsx_filename after success1**  
  *Type:* Set node  
  *Role:* Confirms Excel filename after successful extraction.

- **Merge - Continue workflow1**  
  *Type:* Merge node  
  *Role:* Merges successful or existing file processing paths.

- **Set Parameters1**  
  *Type:* Set node  
  *Role:* Passes along the Excel filename as path_to_file for reading.

- **Read Excel File1**  
  *Type:* Read Binary File  
  *Role:* Reads the Excel file binary data.

- **Parse Excel1**  
  *Type:* Spreadsheet File node  
  *Role:* Parses Excel data into JSON format for processing.

- **Extract Headers and Data**  
  *Type:* Code node  
  *Role:* Extracts all unique headers, cleans headers (removes type suffixes), maps original to cleaned headers, and samples values to aid AI analysis.

- **AI Analyze All Headers**  
  *Type:* OpenAI Chat node  
  *Role:* Sends headers and sample values to GPT-4 to assign aggregation rules (sum, mean, first) based on field semantics and sample data.

- **Process AI Response1**  
  *Type:* Code node  
  *Role:* Parses AI aggregation rules, merges with default heuristics, and prepares final aggregation rules and grouping parameter.

- **Group Data with AI Rules1**  
  *Type:* Code node  
  *Role:* Groups raw data by the chosen parameter using aggregation rules (sum for quantities, mean for rates, first for categorical) to produce grouped data entries.

- **On the standard 3D View**  
  *Type:* If node  
  *Role:* Filters data to include only elements visible in the standard 3D view.  
  *Failure path:* Routes non-3D view elements to a separate output.

- **Find Category Fields1**  
  *Type:* Code node  
  *Role:* Analyzes grouped data headers to detect category fields (e.g., Category, IFC Type) and volumetric fields (volume, area, length, count, etc.). Extracts unique category values for classification.  
  *Outputs:* Category field name, type, unique values, volumetric fields, grouped data array.

**Edge Cases:**  
- Missing or malformed Excel file causes extraction or parsing failure.  
- AI aggregation rule response may fail JSON parsing; fallback heuristics apply.  
- Grouping by empty or missing parameter results in partial grouping or empty output.  
- Elements not visible in 3D view are segregated for separate handling.

---

#### Block 2. Element Classification

**Overview:**  
This block uses AI to classify grouped elements as building elements or non-building elements (e.g., annotations, drawings), enabling further focused analysis only on relevant building components.

**Nodes Involved:**  
- AI Classify Categories1  
- Apply Classification to Groups1  
- Is Building Element1  
- Non-Building Elements Output1  

**Node Details:**

- **AI Classify Categories1**  
  *Type:* OpenAI Chat node (GPT-4)  
  *Role:* Classifies category values as building elements (true) or non-building elements (false) using an expert system prompt focused on BIM and Revit domain knowledge.  
  *Parameters:* Uses low temperature (0.1) for deterministic classification, returns JSON with classification mappings and lists of building/drawing categories.  
  *Inputs:* Category field and its values.  
  *Outputs:* JSON with classifications.

- **Apply Classification to Groups1**  
  *Type:* Code node  
  *Role:* Parses AI classification JSON, applies classification to each group by category value.  
  *Logic:*  
    - If AI classification exists for category value, use it with high confidence (95%).  
    - Else, fallback to keyword matching on category strings to infer building vs. non-building elements with moderate confidence (85%).  
    - If no category field, infer from volumetric data presence.  
  *Outputs:* Groups enriched with is_building_element flag, confidence, and reason.

- **Is Building Element1**  
  *Type:* If node  
  *Role:* Splits flow based on building element flag:  
    - True: proceeds to material analysis.  
    - False: routed to non-building elements output.

- **Non-Building Elements Output1**  
  *Type:* Set node  
  *Role:* Outputs a message indicating non-building elements like annotations and drawings.  
  *Outputs:* Message string.

**Edge Cases:**  
- AI classification JSON parsing errors handled gracefully with console errors.  
- Categories not recognized by AI or keywords default to building element assumption with lower confidence.  
- Groups without category fields rely purely on volumetric data, which may be incomplete.

---

#### Block 3. Material Analysis & Price Estimation

**Overview:**  
Processes building elements in batches to enable scalable AI analysis. Uses Anthropic Claude LLM for detailed material classification, unit determination, and price estimation based on multiple standards and real-time market insights. Accumulates results for aggregation.

**Nodes Involved:**  
- Process in Batches1  
- Clean Empty Values1  
- Prepare Enhanced Prompts  
- Anthropic Chat Model1  
- AI Agent Enhanced  
- Parse Enhanced Response  
- Accumulate Results  
- Check If All Batches Done  
- Collect All Results  

**Node Details:**

- **Process in Batches1**  
  *Type:* SplitInBatches node  
  *Role:* Splits building element groups into single-item batches for iterative AI processing.  
  *Batch Size:* 1

- **Clean Empty Values1**  
  *Type:* Code node  
  *Role:* Cleans empty or null values from batch data to avoid AI confusion.  
  *Logic:* Removes nulls, empty strings, zero values except for 'Element Count'.

- **Prepare Enhanced Prompts**  
  *Type:* Code node  
  *Role:* Generates detailed system and user prompts for AI pricing analysis, incorporating country-specific references, material classification standards (EU, German, US), quantity analysis, price estimation rules, and output JSON schema.  
  *Inputs:* Current batch data and country parameter from setup.

- **Anthropic Chat Model1**  
  *Type:* Anthropic Claude Chat node  
  *Role:* Executes the AI analysis with the prepared prompts, expecting detailed JSON output with material classification, quantity, price, and confidence data.

- **AI Agent Enhanced**  
  *Type:* Langchain Agent node  
  *Role:* Orchestrates the AI interaction using the prompt and model response.

- **Parse Enhanced Response**  
  *Type:* Code node  
  *Role:* Parses AI JSON output, enriches original data with classification, pricing, confidence scores, impact category, and metadata. Handles parsing errors gracefully, marking analysis as failed if necessary.

- **Accumulate Results**  
  *Type:* Code node  
  *Role:* Stores each processed batch result into global static data for later aggregation. Logs accumulation status.

- **Check If All Batches Done**  
  *Type:* If node  
  *Role:* Checks if batch processing is complete (no items left). If not, loops back to Process in Batches.

- **Collect All Results**  
  *Type:* Code node  
  *Role:* Collects all accumulated results from static data, clears the accumulator for next run, and outputs all processed items.

**Edge Cases:**  
- AI response parsing may fail or return incomplete data; handled with error flags.  
- Batch processing ensures large datasets are handled without timeouts or overload.  
- AI model credentials and rate limits can cause failures or delays.

---

#### Block 4. Price Calculation & Reporting

**Overview:**  
Aggregates all processed data to compute project totals, material and category summaries, and generates professional reports both as HTML with charts and as multi-sheet Excel files.

**Nodes Involved:**  
- Calculate Project Totals1  
- Prepare Excel Data  
- Create Excel File  
- Enhance Excel Output  
- Prepare Excel Path1  
- Write Excel to Project Folder1  
- Generate HTML Report  
- Convert to Binary  
- Prepare HTML Path  
- Write HTML to Project Folder  
- Open HTML in Browser  

**Node Details:**

- **Calculate Project Totals1**  
  *Type:* Code node  
  *Role:* Aggregates all processed items to calculate totals by project, material, category, and impact. Computes percentages and rankings for cost and element counts.

- **Prepare Excel Data**  
  *Type:* Code node  
  *Role:* Formats aggregated and detailed data into multiple sheets suitable for Excel export: Summary, Detailed Elements, Material Summary, Top 10 Expensive Groups.

- **Create Excel File**  
  *Type:* Spreadsheet File node  
  *Role:* Converts prepared JSON data into an XLSX file with multiple sheets.

- **Enhance Excel Output**  
  *Type:* Code node  
  *Role:* Adds metadata, file name, mime type, and other properties to the Excel binary data.

- **Prepare Excel Path1**  
  *Type:* Code node  
  *Role:* Defines the final path and filename for saving the Excel report based on the project folder and current date.

- **Write Excel to Project Folder1**  
  *Type:* Write Binary File node  
  *Role:* Writes the Excel report to disk.

- **Generate HTML Report**  
  *Type:* Code node  
  *Role:* Creates a professional, styled HTML report with embedded Chart.js visualizations including pie charts, bar charts, Pareto analysis, KPIs, and key insights.  
  *Features:*  
    - Uses project metadata and calculated totals  
    - Responsive design with print styles  
    - Interactive charts with tooltips  
    - Sections include executive summary, analytics, methodology, and footer with resources.

- **Convert to Binary**  
  *Type:* Code node  
  *Role:* Converts generated HTML string to base64 binary for writing.

- **Prepare HTML Path**  
  *Type:* Code node  
  *Role:* Determines final file path for saving the HTML report.

- **Write HTML to Project Folder**  
  *Type:* Write Binary File node  
  *Role:* Saves the HTML report to disk.

- **Open HTML in Browser**  
  *Type:* Execute Command node  
  *Role:* Opens the saved HTML file in the default web browser for immediate viewing.

**Edge Cases:**  
- File system write permissions may cause errors saving reports.  
- Large data sets may impact performance in report generation and rendering.  
- Chart.js dependency is loaded externally; offline usage requires adjustment.

---

#### Conversion Block

**Overview:**  
Ensures data availability by checking for existing Excel extraction of the Revit file. If missing, runs an external Revit converter executable to produce the Excel file for downstream processing.

**Nodes Involved:**  
- Check - Does Excel file exist?1 (shared with Block 1)  
- If - File exists?1 (shared with Block 1)  
- Extract - Run converter1 (shared with Block 1)  
- Check - Did extraction succeed?1 (shared with Block 1)  
- Error - Show what went wrong1 (shared with Block 1)  
- Info - Skip conversion1  
- Merge - Continue workflow1  

**Node Details:**

- **Info - Skip conversion1**  
  *Type:* Set node  
  *Role:* Sets status message when Excel file already exists, skipping conversion.

- **Merge - Continue workflow1**  
  *Type:* Merge node  
  *Role:* Merges both conversion and skip paths to continue.

---

### 3. Summary Table

| Node Name                   | Node Type                         | Functional Role                                     | Input Node(s)                  | Output Node(s)                      | Sticky Note                                                                                                             |
|-----------------------------|----------------------------------|----------------------------------------------------|--------------------------------|------------------------------------|-------------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger                   | Workflow entry point                               |                                | Setup - Define file paths           |                                                                                                                         |
| Setup - Define file paths    | Set                              | Defines paths, project file, grouping, country     | When clicking ‘Execute workflow’ | Create - Excel filename1            |                                                                                                                         |
| Create - Excel filename1     | Set                              | Builds expected Excel filename                      | Setup - Define file paths        | Check - Does Excel file exist?1     |                                                                                                                         |
| Check - Does Excel file exist?1 | Read Binary File                 | Checks if Excel file exists                         | Create - Excel filename1         | If - File exists?1                  |                                                                                                                         |
| If - File exists?1           | If                               | Branch based on file existence                      | Check - Does Excel file exist?1  | Info - Skip conversion1 / Extract - Run converter1 |                                                                                                                         |
| Info - Skip conversion1      | Set                              | Status message if file exists, skip conversion     | If - File exists?1               | Merge - Continue workflow1          |                                                                                                                         |
| Extract - Run converter1     | Execute Command                  | Runs converter if Excel file missing                | If - File exists?1               | Check - Did extraction succeed?1    |                                                                                                                         |
| Check - Did extraction succeed?1 | If                               | Verifies success of conversion                      | Extract - Run converter1         | Error - Show what went wrong1 / Set xlsx_filename after success1 |                                                                                                                         |
| Error - Show what went wrong1 | Set                              | Captures and logs extraction errors                | Check - Did extraction succeed?1 | Merge - Continue workflow1          |                                                                                                                         |
| Set xlsx_filename after success1 | Set                              | Sets Excel filename after successful conversion    | Check - Did extraction succeed?1 | Merge - Continue workflow1          |                                                                                                                         |
| Merge - Continue workflow1   | Merge                            | Merges skip and success paths                       | Info - Skip conversion1 / Error - Show what went wrong1 / Set xlsx_filename after success1 | Set Parameters1                    |                                                                                                                         |
| Set Parameters1             | Set                              | Passes Excel filename as path_to_file               | Merge - Continue workflow1       | Read Excel File1                   |                                                                                                                         |
| Read Excel File1            | Read Binary File                 | Reads Excel file binary data                         | Set Parameters1                 | Parse Excel1                      |                                                                                                                         |
| Parse Excel1                | Spreadsheet File                 | Parses Excel into JSON                               | Read Excel File1                | Extract Headers and Data           |                                                                                                                         |
| Extract Headers and Data    | Code                             | Cleans headers, maps, samples values                | Parse Excel1                   | AI Analyze All Headers             |                                                                                                                         |
| AI Analyze All Headers      | OpenAI Chat                      | Determines aggregation rules for each header        | Extract Headers and Data        | Process AI Response1               |                                                                                                                         |
| Process AI Response1        | Code                             | Parses AI response, merges with heuristics          | AI Analyze All Headers          | Group Data with AI Rules1          |                                                                                                                         |
| Group Data with AI Rules1   | Code                             | Groups raw data by parameter with aggregation rules | Process AI Response1            | On the standard 3D View            |                                                                                                                         |
| On the standard 3D View     | If                               | Filters elements visible in 3D view                  | Group Data with AI Rules1       | Find Category Fields1 / Non-3D View Elements Output |                                                                                                                         |
| Find Category Fields1       | Code                             | Detects category and volumetric fields               | On the standard 3D View         | AI Classify Categories1            |                                                                                                                         |
| AI Classify Categories1     | OpenAI Chat                      | Classifies categories as building/non-building       | Find Category Fields1           | Apply Classification to Groups1    |                                                                                                                         |
| Apply Classification to Groups1 | Code                             | Applies classification, assigns confidence           | AI Classify Categories1         | Is Building Element1               |                                                                                                                         |
| Is Building Element1        | If                               | Splits flow by building element flag                 | Apply Classification to Groups1 | Process in Batches1 / Non-Building Elements Output1 |                                                                                                                         |
| Non-Building Elements Output1 | Set                              | Outputs message for non-building elements            | Is Building Element1 (False)    |                                |                                                                                                                         |
| Process in Batches1         | SplitInBatches                   | Processes building elements in individual batches    | Is Building Element1 (True)     | Clean Empty Values1                |                                                                                                                         |
| Clean Empty Values1         | Code                             | Cleans batch data removing empty/null values         | Process in Batches1             | Prepare Enhanced Prompts           |                                                                                                                         |
| Prepare Enhanced Prompts    | Code                             | Builds detailed AI prompts for price estimation      | Clean Empty Values1             | AI Agent Enhanced                 |                                                                                                                         |
| Anthropic Chat Model1       | Anthropic Claude Chat            | AI model for material classification and pricing     | Prepare Enhanced Prompts        | AI Agent Enhanced (ai_languageModel) |                                                                                                                         |
| AI Agent Enhanced           | Langchain Agent                  | Runs AI analysis agent with prepared prompts          | Anthropic Chat Model1           | Parse Enhanced Response           |                                                                                                                         |
| Parse Enhanced Response     | Code                             | Parses AI output, enriches results, handles errors   | AI Agent Enhanced              | Accumulate Results                |                                                                                                                         |
| Accumulate Results         | Code                             | Accumulates processed batch results globally         | Parse Enhanced Response         | Check If All Batches Done          |                                                                                                                         |
| Check If All Batches Done  | If                               | Checks completion of batch processing                  | Accumulate Results             | Collect All Results / Process in Batches1 |                                                                                                                         |
| Collect All Results        | Code                             | Collects all accumulated batch results                 | Check If All Batches Done (True) | Calculate Project Totals1          |                                                                                                                         |
| Calculate Project Totals1  | Code                             | Aggregates project-wide totals and rankings            | Collect All Results             | Prepare Excel Data / Generate HTML Report | Sticky Note: Block 4 summary (Price Calculation & Reporting)                                                            |
| Prepare Excel Data         | Code                             | Formats data for multi-sheet Excel export              | Calculate Project Totals1       | Create Excel File                 |                                                                                                                         |
| Create Excel File          | Spreadsheet File                 | Creates XLSX file with multiple sheets                  | Prepare Excel Data              | Enhance Excel Output              |                                                                                                                         |
| Enhance Excel Output       | Code                             | Adds metadata and file info to Excel binary             | Create Excel File              | Prepare Excel Path1              |                                                                                                                         |
| Prepare Excel Path1        | Code                             | Determines final Excel file save path                   | Enhance Excel Output           | Write Excel to Project Folder1    |                                                                                                                         |
| Write Excel to Project Folder1 | Write Binary File               | Saves Excel report to disk                               | Prepare Excel Path1            |                                |                                                                                                                         |
| Generate HTML Report       | Code                             | Builds professional HTML report with charts             | Calculate Project Totals1       | Convert to Binary                |                                                                                                                         |
| Convert to Binary          | Code                             | Converts HTML to base64 binary for saving                | Generate HTML Report           | Prepare HTML Path                |                                                                                                                         |
| Prepare HTML Path          | Code                             | Defines file path for HTML report save                   | Convert to Binary              | Write HTML to Project Folder      |                                                                                                                         |
| Write HTML to Project Folder | Write Binary File               | Saves HTML report to disk                                | Prepare HTML Path              | Open HTML in Browser             |                                                                                                                         |
| Open HTML in Browser       | Execute Command                 | Opens the saved HTML report in default browser          | Write HTML to Project Folder   |                                |                                                                                                                         |
| Sticky Note                | Sticky Note                     | Provides GitHub star encouragement                       |                                |                                | ⭐ **If you find our tools helpful**, please consider **starring our repository** on [GitHub](https://github.com/datadrivenconstruction/cad2data-Revit-IFC-DWG-DGN-pipeline-with-conversion-validation-qto). Your support helps us improve and continue developing open solutions for the community! |
| Sticky Note (Element Classification Block) | Sticky Note                     | Describes Block 2: Element Classification               |                                |                                | ## Block 2: Element Classification ... (see section 1.2 for details)                                                    |
| Sticky Note (Material Analysis Block) | Sticky Note                     | Describes Block 3: Material Analysis                      |                                |                                | ## Block 3: Material Analysis ... (see section 1.3 for details)                                                          |
| Sticky Note (Price Analysis Block) | Sticky Note                     | Describes Block 4: Price Calculation & Reporting          |                                |                                | ## Block 4: Price Calculation & Reporting ... (see section 1.4 for details)                                              |
| Sticky Note (Conversion Block) | Sticky Note                     | Describes Conversion Block: Excel extraction from Revit    |                                |                                | ## Conversion Block ... (see section above)                                                                              |
| Sticky Note (Important Notes) | Sticky Note                     | Important instructions and credits                        |                                |                                | ## ⚠️ Important Information ... Pipeline uses AI, Revit converter requirements, aggregation notes, report formats        |
| Sticky Note2                | Sticky Note                     | Instructions to only modify variables in Setup node       |                                |                                | ## ⬇️ Only modify the variables here  — everything else works automatically                                              |
| Setup Instructions          | Sticky Note                     | Setup detailed instructions                                |                                |                                | ## 📝 Setup Instructions ... Specify paths, grouping, country, and API keys                                              |
| Project Cost Calculation for Revit and IFC | Sticky Note                     | Project title and GitHub link                              |                                |                                | # Project Cost Calculation for Revit and IFC with AI (LLM) DataDrivenConstruction [GitHub](https://github.com/datadrivenconstruction/cad2data-Revit-IFC-DWG-DGN-pipeline-with-conversion-validation-qto) |

---

### 4. Reproducing the Workflow from Scratch

**Step 1: Setup Entry and Parameters**  
1. Create a **Manual Trigger** node named “When clicking ‘Execute workflow’”.  
2. Create a **Set** node named “Setup - Define file paths” connected to the trigger. Define the following parameters:  
   - `path_to_converter`: Full path to the Revit converter executable (e.g., `RvtExporter.exe`)  
   - `project_file`: Full path to the Revit `.rvt` file  
   - `group_by`: String parameter for grouping (e.g., `"Type Name"`)  
   - `country`: Target country for price localization (e.g., `"Germany"`)

**Step 2: Excel Extraction Check and Conversion**  
3. Add a **Set** node “Create - Excel filename1” to generate the expected Excel filename based on the project file name.  
4. Add a **Read Binary File** node “Check - Does Excel file exist?1” to check for the Excel file presence, configured to continue on fail.  
5. Add an **If** node “If - File exists?1” that checks if the binary data exists (file exists).  
6. On the true branch, connect to a **Set** node “Info - Skip conversion1” that sets a status message and passes the filename.  
7. On the false branch, add an **Execute Command** node “Extract - Run converter1” to run the converter executable with the project file as argument.  
8. Add an **If** node “Check - Did extraction succeed?1” checking for absence of error in the converter output.  
9. On failure, connect to a **Set** node “Error - Show what went wrong1” to capture error messages.  
10. On success, connect to a **Set** node “Set xlsx_filename after success1” to set the Excel filename.  
11. Merge all three paths (skip, error, success) with a **Merge** node “Merge - Continue workflow1”.

**Step 3: Excel File Reading and Parsing**  
12. Connect “Merge - Continue workflow1” to a **Set** node “Set Parameters1” that sets `path_to_file` to the Excel filename.  
13. Add a **Read Binary File** node “Read Excel File1” to read the Excel file.  
14. Add a **Spreadsheet File** node “Parse Excel1” to parse the Excel data into JSON.

**Step 4: Header Extraction and AI Aggregation Rules**  
15. Add a **Code** node “Extract Headers and Data” to:  
    - Extract unique headers, clean header names, map originals to cleaned  
    - Sample values for AI context  
16. Add an **OpenAI Chat** node “AI Analyze All Headers” with GPT-4 configured:  
    - System prompt instructs aggregation rules assignment (sum, mean, first) based on header names and sample values  
    - Pass cleaned headers and samples as user prompt  
17. Add a **Code** node “Process AI Response1” to parse AI JSON, merge with heuristic defaults, and prepare final aggregation rules and grouping parameter.

**Step 5: Grouping Data**  
18. Add a **Code** node “Group Data with AI Rules1” that groups raw data by the chosen parameter using the aggregation rules.

**Step 6: Filter by 3D View and Category Field Detection**  
19. Add an **If** node “On the standard 3D View” to filter elements where parameter `On the standard 3D View` is true.  
20. On true branch, connect to a **Code** node “Find Category Fields1” to detect category fields and volumetric fields and prepare data for classification.

**Step 7: AI Classification of Categories**  
21. Add an **OpenAI Chat** node “AI Classify Categories1” with GPT-4:  
    - System prompt specifying BIM expert classification into building/non-building elements  
    - Input category field values from “Find Category Fields1”  
22. Add a **Code** node “Apply Classification to Groups1” to apply AI classifications or keyword-based fallback logic, assign confidence and reasons.

**Step 8: Split Flow by Building Elements**  
23. Add an **If** node “Is Building Element1” to route:  
    - True: building elements to batch processing  
    - False: to **Set** node “Non-Building Elements Output1” with message.

**Step 9: Batch Processing and AI Price Analysis**  
24. Add a **SplitInBatches** node “Process in Batches1” with batch size 1.  
25. Add a **Code** node “Clean Empty Values1” to clean null/empty fields from batch data.  
26. Add a **Code** node “Prepare Enhanced Prompts” that prepares detailed system and user prompts for AI pricing analysis including material classification and price estimation schema.  
27. Add an **Anthropic Chat** node “Anthropic Chat Model1” configured with Claude model and credentials connected to a **Langchain Agent** node “AI Agent Enhanced”.  
28. Connect “AI Agent Enhanced” to a **Code** node “Parse Enhanced Response” to parse AI JSON output and enrich original data with classifications, prices, confidence, and impact category.  
29. Add a **Code** node “Accumulate Results” that appends each batch result into workflow static data.  
30. Add an **If** node “Check If All Batches Done” checking a flag from batch context to loop or continue.  
31. On completion, add a **Code** node “Collect All Results” to gather all accumulated results and clear the accumulator.

**Step 10: Aggregation and Reporting**  
32. Add a **Code** node “Calculate Project Totals1” to aggregate project totals, cost distributions, and rankings.  
33. Add a **Code** node “Prepare Excel Data” to format data into multiple sheets (summary, detailed, material, top groups).  
34. Add a **Spreadsheet File** node “Create Excel File” to generate XLSX file.  
35. Add a **Code** node “Enhance Excel Output” adding metadata and file properties.  
36. Add a **Code** node “Prepare Excel Path1” to determine the save path with timestamp.  
37. Add a **Write Binary File** node “Write Excel to Project Folder1” to save the Excel report.  
38. Add a **Code** node “Generate HTML Report” to build a full-featured HTML report with Chart.js visualizations and styled sections.  
39. Add a **Code** node “Convert to Binary” to encode the HTML string for saving.  
40. Add a **Code** node “Prepare HTML Path” to generate the save path with timestamp.  
41. Add a **Write Binary File** node “Write HTML to Project Folder” to save the HTML report.  
42. Add an **Execute Command** node “Open HTML in Browser” to open the report immediately.

**Step 11: Add Sticky Notes**  
43. Add sticky note nodes to annotate each block with descriptions and instructions as per provided content.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                              | Context or Link                                                                                                      |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------|
| ⭐ **If you find our tools helpful**, please consider **starring our repository** on [GitHub](https://github.com/datadrivenconstruction/cad2data-Revit-IFC-DWG-DGN-pipeline-with-conversion-validation-qto). Your support helps us improve and continue developing open solutions for the community! | GitHub repository for the pipeline                                                                                   |
| Pipeline uses AI (OpenAI, Grok, Anthropic) - check credits and limits.                                                                                                                     | Important usage considerations                                                                                        |
| Revit converter requires downloaded DDC_Converter_Revit executable for Excel extraction.                                                                                                   | External dependency                                                                                                  |
| Data is aggregated by groups; volumes and areas are already aggregated totals; do not multiply by element count again.                                                                       | Data handling note                                                                                                   |
| Reports are generated in HTML and Excel formats for detailed analysis and visualization.                                                                                                   | Output formats                                                                                                       |
| Setup instructions: specify path to converter executable, project file, grouping parameter, country, and ensure proper API credentials for AI nodes.                                       | Setup - Define file paths sticky note                                                                                 |
| The workflow includes fallback heuristics for aggregation rules and classification, ensuring robustness against AI parsing failures.                                                      | Resilience considerations                                                                                            |
| The HTML report includes embedded Chart.js visualizations and is designed with a professional style inspired by McKinsey/Accenture, with responsive and print-friendly layouts.             | Report styling and visualization                                                                                     |
| The AI prompts incorporate domain-specific knowledge about BIM, construction materials, multi-standard classifications, and pricing strategies, ensuring high relevance and accuracy.       | AI prompt design                                                                                                     |

---

**Disclaimer:**  
The provided description is automatically generated from an n8n workflow JSON export. This workflow respects all content policies, contains no illegal or protected content, and manipulates only public and legal data sources.