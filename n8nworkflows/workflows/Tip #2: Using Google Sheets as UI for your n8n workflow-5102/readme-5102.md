Tip #2: Using Google Sheets as UI for your n8n workflow

https://n8nworkflows.xyz/workflows/tip--2--using-google-sheets-as-ui-for-your-n8n-workflow-5102


# Tip #2: Using Google Sheets as UI for your n8n workflow

---

### 1. Workflow Overview

This workflow demonstrates how to use **Google Sheets as a user interface (UI)** for controlling an n8n workflow’s input and output, effectively adding a layer of interactivity and control via a spreadsheet. It is designed to process rows marked as "READY" in a Google Sheet, execute logic on each row, and update the sheet to reflect the workflow’s progress and results.

The workflow is logically divided into three main blocks:

- **1.1 Input Reception:** Initiates the workflow manually and reads data from Google Sheets filtered by status.
- **1.2 Iterative Processing:** Processes each qualifying row individually using batching and data transformation.
- **1.3 Output Update:** Updates Google Sheets rows to mark them as processed, and appends additional computed data.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
This block triggers the workflow manually and reads rows from a Google Sheet where the "Status" column equals "READY". It serves as the entry point and filters input data for processing.

**Nodes Involved:**  
- When clicking ‘Execute workflow’  
- Read Google Sheets

**Node Details:**

- **When clicking ‘Execute workflow’**  
  - Type: Manual Trigger  
  - Role: Starts the workflow on user demand  
  - Config: No parameters needed; straightforward manual trigger  
  - Inputs: None  
  - Outputs: Connected to "Read Google Sheets"  
  - Edge Cases: None typical, but workflow will not run unless manually triggered  
  - Version: 1  

- **Read Google Sheets**  
  - Type: Google Sheets Node (Read operation)  
  - Role: Reads rows from the spreadsheet filtered by "Status" = "READY"  
  - Config:  
    - Document ID: Spreadsheet titled "Tip #2: Using Google Sheets as UI for your n8n workflow"  
    - Sheet: "Sheet1" (gid=0)  
    - Filter: Retrieve only rows where the "Status" column equals "READY"  
    - Authentication: Uses a Google Service Account credential  
  - Inputs: Receives trigger from manual node  
  - Outputs: Sends data to "Loop Over Items"  
  - Edge Cases:  
    - API errors (auth issues, quota limits)  
    - No rows match filter (empty dataset)  
    - Service account lacking permissions  
  - Version: 4.6  

---

#### 1.2 Iterative Processing

**Overview:**  
This block processes each filtered row individually by splitting the data into batches (default batch size = 1), modifying fields, and preparing data for update.

**Nodes Involved:**  
- Loop Over Items  
- Edit Fields

**Node Details:**

- **Loop Over Items**  
  - Type: SplitInBatches  
  - Role: Processes input rows one at a time for granular control  
  - Config: Default batch size (process one item per iteration)  
  - Inputs: Receives filtered rows from "Read Google Sheets"  
  - Outputs: Two outputs connected downstream: one to "Edit Fields" and another to "Update Google Sheets"  
  - Edge Cases: Performance impact for large datasets if batch size is too small or too large  
  - Version: 3  

- **Edit Fields**  
  - Type: Set Node  
  - Role: Modifies data fields for the current batch item before updating Google Sheets  
  - Config Assignments:  
    - `row_number`: Copies existing row number from input JSON (used for matching rows in update)  
    - `Number`: Sets to the length of the string in the `Color` field of the row (`={{ $json.Color.length }}`)  
    - `Status`: Sets to "DONE" to mark processing completion  
  - Inputs: Receives one row at a time from "Loop Over Items"  
  - Outputs: Connects back to "Loop Over Items" (loop output 1) and ultimately to "Update Google Sheets"  
  - Edge Cases:  
    - Missing or null `Color` field causing expression errors  
    - Row number missing or invalid (would break update matching)  
  - Version: 3.4  

---

#### 1.3 Output Update

**Overview:**  
This block updates the Google Sheets rows with modified data, marking each processed row as "DONE", updating the "Number" field with the length of the "Color" string, and ensuring rows are matched by their row number.

**Nodes Involved:**  
- Update Google Sheets

**Node Details:**

- **Update Google Sheets**  
  - Type: Google Sheets Node (Update operation)  
  - Role: Writes back updated data to the original spreadsheet  
  - Config:  
    - Document ID and Sheet same as in "Read Google Sheets"  
    - Matching Column: `row_number` used to identify the exact row to update  
    - Columns Updated: `Color`, `Status`, `Number`, `row_number` (readonly but included for matching)  
    - Mapping Mode: Automatic based on input data  
    - Authentication: Google Service Account  
  - Inputs: Receives processed row data from the "Loop Over Items" (main output)  
  - Outputs: None (end of processing)  
  - Edge Cases:  
    - Row number mismatch causing update failures  
    - API errors or permission issues  
    - Concurrent updates or rate limits on Google Sheets API  
  - Version: 4.6  

---

### 3. Summary Table

| Node Name                  | Node Type             | Functional Role                        | Input Node(s)                 | Output Node(s)               | Sticky Note                                                                                                                    |
|----------------------------|-----------------------|-------------------------------------|------------------------------|-----------------------------|-------------------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger        | Starts workflow manually             | None                         | Read Google Sheets            |                                                                                                                               |
| Read Google Sheets          | Google Sheets (Read)   | Reads rows where Status = READY      | When clicking ‘Execute workflow’ | Loop Over Items              |                                                                                                                               |
| Loop Over Items             | SplitInBatches         | Processes rows one by one            | Read Google Sheets            | Edit Fields, Update Google Sheets |                                                                                                                               |
| Edit Fields                | Set Node              | Updates fields before writing back   | Loop Over Items               | Loop Over Items              |                                                                                                                               |
| Update Google Sheets        | Google Sheets (Update) | Writes back updated data to sheet    | Loop Over Items               | None                        |                                                                                                                               |
| Sticky Note                | Sticky Note            | Informative note with blog link      | None                         | None                        | ## Tip #2: Using Google Sheets as UI for your n8n workflow\nThis is the template with example of how you can add more control on input and output of your n8n workflow. In other words how to add some kind of UI to your n8n workflow.\nMore details in my [N8n Tips blog](https://n8n-tips.blogspot.com/2025/06/tip-2-using-google-sheets-as-ui-for.html) |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger node**  
   - Name: `When clicking ‘Execute workflow’`  
   - Purpose: To manually start the workflow without any input parameters  

2. **Add a Google Sheets node (Read operation)**  
   - Name: `Read Google Sheets`  
   - Set operation to `Read`  
   - Connect credential: Google Service Account with access to your target Google Sheet  
   - Document ID: Paste your Google Sheets document ID (e.g., `13xu9zKI8yDqs7971qIq9zDCBCnhYzdVq8pdtnQ9vMkI`)  
   - Sheet Name: Use sheet with gid=0 (usually "Sheet1")  
   - Enable Filter: Set to retrieve rows where `Status` column equals `READY`  
   - Connect input from `When clicking ‘Execute workflow’` node  

3. **Add a SplitInBatches node**  
   - Name: `Loop Over Items`  
   - Purpose: To process rows individually  
   - Batch Size: Default (1)  
   - Connect input from `Read Google Sheets` node  

4. **Add a Set node**  
   - Name: `Edit Fields`  
   - Purpose: To modify row fields before updating  
   - Add three fields with expressions:  
     - `row_number` (Number type): `={{ $json.row_number }}`  
     - `Number` (Number type): `={{ $json.Color.length }}` (length of string in `Color`)  
     - `Status` (String type): `DONE`  
   - Connect input from `Loop Over Items` node  
   - Connect output back to `Loop Over Items` node (second output)  

5. **Add a Google Sheets node (Update operation)**  
   - Name: `Update Google Sheets`  
   - Set operation to `Update`  
   - Use same credential, Document ID, and Sheet name as in the read node  
   - Configure columns for update: `Color`, `Status`, `Number`, and `row_number`  
   - Set matching column to `row_number` for identifying rows to update  
   - Connect input from the main output of `Loop Over Items` node  

6. **Add a Sticky Note node (Optional)**  
   - Name: `Sticky Note`  
   - Content:  
     ```
     ## Tip #2: Using Google Sheets as UI for your n8n workflow
     This is the template with example of how you can add more control on input and output of your n8n workflow. In other words how to add some kind of UI to your n8n workflow.
     More details in my [N8n Tips blog](https://n8n-tips.blogspot.com/2025/06/tip-2-using-google-sheets-as-ui-for.html)
     ```  
   - Place visually above the workflow for clarity  

7. **Finalize connections and test**  
   - Verify all credentials are correctly configured and authorized  
   - Test by marking some rows as `READY` in the Google Sheet and executing the workflow manually  
   - Confirm rows update their `Status` to `DONE` and have `Number` set to the length of their `Color` value  

---

### 5. General Notes & Resources

| Note Content                                                                 | Context or Link                                                                                     |
|------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| This workflow is a practical example of using Google Sheets as a UI for controlling n8n workflows, allowing users to manage workflow execution and data input/output through a familiar spreadsheet interface. | Workflow concept explanation                                                                    |
| More details and usage examples can be found on the author’s blog post: [N8n Tips blog](https://n8n-tips.blogspot.com/2025/06/tip-2-using-google-sheets-as-ui-for.html) | Author’s external blog for extended learning and context                                          |

---

**Disclaimer:**  
The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with applicable content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.

---