Google Sheets Duplication & Enrichment Automation

https://n8nworkflows.xyz/workflows/google-sheets-duplication---enrichment-automation-4865


# Google Sheets Duplication & Enrichment Automation

### 1. Workflow Overview

This workflow automates the duplication of all sheets from a master Google Spreadsheet into a newly created destination spreadsheet. It is designed to simplify replicating complex spreadsheets with multiple sheets while preserving sheet names and data structure.  

**Target Use Cases:**  
- Backing up or versioning master spreadsheets  
- Cloning templates for new projects  
- Bulk copying of multi-sheet spreadsheets without manual intervention  

**Logical Blocks:**  
- **1.1 Start & Create:** Trigger and create the new destination spreadsheet.  
- **1.2 Sheet Discovery:** Fetch metadata of the master spreadsheet and extract sheet names and IDs.  
- **1.3 Loop Processing:** Iterate over each sheet individually to prevent API overload and manage creation and copying.  
- **1.4 Sheet Operations:** Create corresponding sheets in the new spreadsheet, read data from the master, and write it to the new sheets.  
- **1.5 Cleanup:** Remove the default "Sheet1" created by Google Sheets in the new spreadsheet after all sheets are copied.

---

### 2. Block-by-Block Analysis

#### 1.1 Start & Create

**Overview:**  
This block initiates the workflow manually and creates a new Google Spreadsheet that will serve as the destination for duplicated sheets.  

**Nodes Involved:**  
- When clicking 'Test workflow' (manual trigger)  
- Create New Spreadsheet  

**Node Details:**  

- **When clicking 'Test workflow'**  
  - Type: Manual Trigger  
  - Role: Allows manual start of the workflow by user interaction.  
  - Configuration: No parameters; triggers workflow on user click.  
  - Inputs: None  
  - Outputs: Connects to Create New Spreadsheet node  
  - Edge Cases: User must manually trigger; no scheduling or automation.  

- **Create New Spreadsheet**  
  - Type: Google Sheets node (resource: spreadsheet, operation: create)  
  - Role: Creates a new spreadsheet to copy sheets into.  
  - Configuration: Title preset to "Copy of Master Spreadsheet" (modifiable)  
  - Inputs: Triggered by manual start node  
  - Outputs: Provides new spreadsheetId and spreadsheetUrl to downstream nodes  
  - Credentials: Requires Google Sheets OAuth2 credentials with spreadsheet creation permissions  
  - Edge Cases: API rate limits, OAuth token expiration, insufficient permissions  
  - Notes: Stores new spreadsheet ID for subsequent sheet creation  

---

#### 1.2 Sheet Discovery

**Overview:**  
Retrieves metadata about the master spreadsheet, specifically the list of sheets, to prepare for duplication.  

**Nodes Involved:**  
- HTTP Request  
- Code  

**Node Details:**  

- **HTTP Request**  
  - Type: HTTP Request node  
  - Role: Calls Google Sheets API v4 to fetch the master spreadsheet metadata  
  - Configuration:  
    - URL: `https://sheets.googleapis.com/v4/spreadsheets/YOUR_MASTER_SPREADSHEET_ID` (replace placeholder)  
    - Headers: Accept application/json  
    - Authentication: Google Sheets OAuth2 credentials  
  - Inputs: Output from Create New Spreadsheet  
  - Outputs: JSON response with sheets metadata  
  - Credentials: Google Sheets OAuth2 credentials  
  - Edge Cases: Invalid spreadsheet ID, authorization errors, API rate limits  

- **Code**  
  - Type: Code node (JavaScript)  
  - Role: Parses API response to extract an array of sheet names, sheet IDs, and indexes for processing  
  - Configuration: Custom JS code that maps over `sheets` array in API response, returning structured JSON items  
  - Inputs: Output of HTTP Request node  
  - Outputs: Array of sheet metadata objects for iteration  
  - Edge Cases: Unexpected response format, missing properties  

---

#### 1.3 Loop Processing

**Overview:**  
Iterates over each sheet extracted from the master spreadsheet metadata one at a time to avoid API throttling and maintain processing order.  

**Nodes Involved:**  
- Loop Over Items (SplitInBatches)  

**Node Details:**  

- **Loop Over Items**  
  - Type: SplitInBatches node  
  - Role: Processes sheets sequentially, emitting one sheet per iteration  
  - Configuration: Default batch size (1 item per batch)  
  - Inputs: Output array from Code node (sheet metadata)  
  - Outputs:  
    - Output 1: Triggered when all sheets processed (used to trigger deletion of default sheet)  
    - Output 2: Triggers sheet creation and data copy per sheet  
  - Edge Cases: Long processing times with many sheets, API limits if batch size increased  

---

#### 1.4 Sheet Operations

**Overview:**  
For each sheet in the loop, creates a new sheet in the destination spreadsheet, reads data from the master spreadsheet sheet, and writes the data to the new sheet preserving column structure.  

**Nodes Involved:**  
- Create Sheets  
- Read Spreadsheet1  
- Write sheet  

**Node Details:**  

- **Create Sheets**  
  - Type: Google Sheets node (operation: create)  
  - Role: Creates a new sheet with the same name as the master sheet in the destination spreadsheet  
  - Configuration:  
    - Title set dynamically using expression `{{$('Loop Over Items').item.json.sheetName}}`  
    - Document ID set dynamically using new spreadsheet ID from Create New Spreadsheet node  
  - Inputs: Triggered from Loop Over Items output 2  
  - Outputs: Passes to Read Spreadsheet1 node  
  - Credentials: Google Sheets OAuth2 credentials  
  - Edge Cases: Duplicate sheet names causing errors, permission issues  

- **Read Spreadsheet1**  
  - Type: Google Sheets node (operation: read)  
  - Role: Reads data from the corresponding sheet in the master spreadsheet  
  - Configuration:  
    - Sheet name dynamically set via expression from Loop Over Items  
    - Document ID set to master spreadsheet ID (hardcoded)  
  - Inputs: From Create Sheets node  
  - Outputs: Passes data to Write sheet node  
  - Credentials: Google Sheets OAuth2 credentials  
  - Edge Cases: Sheet not found, empty sheets, read permission errors  

- **Write sheet**  
  - Type: Google Sheets node (operation: appendOrUpdate)  
  - Role: Writes data read from master sheet into the newly created sheet in destination spreadsheet  
  - Configuration:  
    - Sheet name dynamically set via expression from Loop Over Items  
    - Document ID set via URL of new spreadsheet  
    - Columns defined explicitly (Column 1 to Column 9) with string types, preserving columns  
    - Mapping mode: defineBelow (explicit columns defined)  
    - Attempt to convert types: false (data written as strings)  
  - Inputs: From Read Spreadsheet1 node  
  - Outputs: Feeds back to Loop Over Items for next iteration  
  - Credentials: Google Sheets OAuth2 credentials  
  - Edge Cases: Column mismatch between master and new sheet, data type conflicts, API limits  

---

#### 1.5 Cleanup

**Overview:**  
After all sheets are duplicated, removes the default "Sheet1" that Google Sheets creates automatically in new spreadsheets to leave a clean copy.  

**Nodes Involved:**  
- Google Sheets2 (Remove Sheet1)  

**Node Details:**  

- **Google Sheets2**  
  - Type: Google Sheets node (operation: remove)  
  - Role: Removes the default "Sheet1" from the new spreadsheet after all sheets are created  
  - Configuration:  
    - Sheet name: "Sheet1" (hardcoded)  
    - Document ID: URL of new spreadsheet from Create New Spreadsheet node  
  - Inputs: Output 1 (done) from Loop Over Items  
  - Outputs: None (end of workflow)  
  - Credentials: Google Sheets OAuth2 credentials  
  - Edge Cases: If "Sheet1" was already removed or renamed, operation will fail silently or error  

---

### 3. Summary Table

| Node Name               | Node Type               | Functional Role                     | Input Node(s)              | Output Node(s)            | Sticky Note                                                                                         |
|-------------------------|-------------------------|-----------------------------------|----------------------------|---------------------------|---------------------------------------------------------------------------------------------------|
| Workflow Overview       | Sticky Note             | Workflow purpose & process summary| None                       | None                      | # 📊 Google Sheets Duplication Workflow\n\n**Purpose:** Duplicate sheets from master spreadsheet to new spreadsheet\n\n**Process Overview:**\n1. Create new spreadsheet\n2. Get all sheets from master\n3. Loop through each sheet\n4. Create sheet in new spreadsheet\n5. Copy data from master to new\n\n**Required Setup:**\n- Google Sheets OAuth credentials\n- Master spreadsheet ID\n- Proper API permissions |
| Start & Create          | Sticky Note             | Manual trigger & new spreadsheet creation | None                       | None                      | ## 🚀 Start & Create\n\n**Manual Trigger:**\n- Click to start workflow\n- No scheduling needed\n\n**Create New Spreadsheet:**\n- Creates destination spreadsheet\n- Update title as needed\n- Stores ID for later use |
| Sheet Discovery         | Sticky Note             | Fetch master spreadsheet sheets metadata | None                       | None                      | ## 🔍 Sheet Discovery\n\n**HTTP Request:**\n- Gets spreadsheet metadata\n- Lists all sheets\n- Uses Google Sheets API v4\n\n**Code Node:**\n- Extracts sheet names\n- Formats for processing\n- Creates sheet array |
| Loop Processing         | Sticky Note             | Process sheets one by one in loop  | None                       | None                      | ## 🔄 Loop Processing\n\n**Split In Batches:**\n- Processes one sheet at a time\n- Prevents API overload\n- Maintains order\n\n**Two outputs:**\n1. Done - Removes default Sheet1\n2. Loop - Creates & copies sheets |
| Sheet Operations        | Sticky Note             | Create sheets, read & write data   | None                       | None                      | ## 📝 Sheet Operations\n\n**Create Sheets:**\n- Creates new sheet in destination\n- Uses original sheet names\n\n**Read & Write:**\n- Reads from master sheet\n- Writes to new sheet\n- Preserves all columns |
| Important Notes         | Sticky Note             | Key operational instructions       | None                       | None                      | ⚠️ **Important Notes:**\n\n1. Replace spreadsheet ID in HTTP Request\n2. Update credentials for all nodes\n3. Default 'Sheet1' is removed after all sheets are created\n4. Column structure must match between sheets |
| When clicking 'Test workflow' | Manual Trigger          | Manual workflow start              | None                       | Create New Spreadsheet    | Manual trigger - click to start the spreadsheet duplication process                                |
| Create New Spreadsheet  | Google Sheets           | Create new destination spreadsheet | When clicking 'Test workflow' | HTTP Request             | Creates new destination spreadsheet - update title as needed                                      |
| HTTP Request            | HTTP Request            | Get master spreadsheet metadata    | Create New Spreadsheet      | Code                      | Gets spreadsheet metadata - replace YOUR_MASTER_SPREADSHEET_ID with actual ID                    |
| Code                    | Code                    | Extract sheet names & IDs           | HTTP Request               | Loop Over Items            | Extracts sheet names and IDs from API response                                                    |
| Loop Over Items         | SplitInBatches          | Iterate sheets one by one           | Code                       | Google Sheets2, Create Sheets | Processes sheets one by one - output 1 when done, output 2 for each iteration                    |
| Create Sheets           | Google Sheets           | Create sheet in new spreadsheet    | Loop Over Items (output 2)  | Read Spreadsheet1         | Creates new sheet in destination spreadsheet with original name                                  |
| Read Spreadsheet1       | Google Sheets           | Read data from master sheet         | Create Sheets              | Write sheet                | Reads data from master spreadsheet - one sheet at a time                                        |
| Write sheet             | Google Sheets           | Write data to new sheet             | Read Spreadsheet1          | Loop Over Items            | Writes data to new sheet - preserves all columns from master                                    |
| Google Sheets2          | Google Sheets           | Remove default "Sheet1"              | Loop Over Items (output 1)  | None                      | Removes default 'Sheet1' after all sheets are created                                            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node:**  
   - Add a Manual Trigger node named "When clicking 'Test workflow'".  
   - No parameters needed. This node starts the workflow on demand.

2. **Create Google Sheets Node to Create New Spreadsheet:**  
   - Add Google Sheets node named "Create New Spreadsheet".  
   - Set Resource to "spreadsheet" and Operation to "create".  
   - Set Title to "Copy of Master Spreadsheet" or desired name.  
   - Connect Manual Trigger output to this node.  
   - Assign Google Sheets OAuth2 credentials with permissions to create spreadsheets.

3. **Create HTTP Request Node to Get Master Spreadsheet Metadata:**  
   - Add HTTP Request node named "HTTP Request".  
   - Set URL to `https://sheets.googleapis.com/v4/spreadsheets/YOUR_MASTER_SPREADSHEET_ID` (replace placeholder).  
   - Set Method to GET (default).  
   - Add Header "Accept" with value "application/json".  
   - Enable Authentication using Google Sheets OAuth2 credentials.  
   - Connect output of "Create New Spreadsheet" to this node.

4. **Create Code Node to Extract Sheets:**  
   - Add Code node named "Code".  
   - Paste JavaScript code to extract sheets:  
     ```javascript
     const spreadsheet = $input.first().json;
     const sheets = spreadsheet.sheets;
     const sheetList = sheets.map(sheet => ({
       json: {
         sheetName: sheet.properties.title,
         sheetId: sheet.properties.sheetId,
         index: sheet.properties.index
       }
     }));
     return sheetList;
     ```  
   - Connect output of "HTTP Request" to this node.

5. **Add SplitInBatches Node to Loop Over Sheets:**  
   - Add SplitInBatches node named "Loop Over Items".  
   - Keep default batch size = 1.  
   - Connect output of "Code" node to this node.  
   - This node will have two outputs:  
     - Output 1 (default): triggers cleanup after processing all sheets  
     - Output 2: triggers sheet creation and data copy per sheet

6. **Create Google Sheets Node to Create Sheets in Destination:**  
   - Add Google Sheets node named "Create Sheets".  
   - Set Operation to "create" under resource "sheet".  
   - Set Title to expression: `{{$('Loop Over Items').item.json.sheetName}}` dynamically.  
   - Set Document ID to new spreadsheet ID from "Create New Spreadsheet" node using expression: `{{$('Create New Spreadsheet').item.json.spreadsheetId}}`.  
   - Connect Output 2 of "Loop Over Items" to this node.  
   - Assign Google Sheets OAuth2 credentials.

7. **Create Google Sheets Node to Read Data from Master Sheet:**  
   - Add Google Sheets node named "Read Spreadsheet1".  
   - Set Operation to "read".  
   - Set Sheet Name dynamically: `{{$('Loop Over Items').item.json.sheetName}}`.  
   - Set Document ID to master spreadsheet ID (static string).  
   - Connect output of "Create Sheets" to this node.  
   - Assign Google Sheets OAuth2 credentials.

8. **Create Google Sheets Node to Write Data to New Sheet:**  
   - Add Google Sheets node named "Write sheet".  
   - Set Operation to "appendOrUpdate".  
   - Set Sheet Name dynamically: `{{$('Loop Over Items').item.json.sheetName}}`.  
   - Set Document ID to new spreadsheet URL from "Create New Spreadsheet" node: `{{$('Create New Spreadsheet').item.json.spreadsheetUrl}}`.  
   - Define columns explicitly (Column 1 to Column 9), all as string type, to match master sheet structure.  
   - Connect output of "Read Spreadsheet1" to this node.  
   - Connect output of "Write sheet" back to "Loop Over Items" to continue iteration.  
   - Assign Google Sheets OAuth2 credentials.

9. **Create Google Sheets Node to Remove Default "Sheet1":**  
   - Add Google Sheets node named "Google Sheets2".  
   - Set Operation to "remove".  
   - Set Sheet Name to "Sheet1" (hardcoded).  
   - Set Document ID to new spreadsheet URL using expression: `{{$('Create New Spreadsheet').item.json.spreadsheetUrl}}`.  
   - Connect Output 1 (done) of "Loop Over Items" to this node.  
   - Assign Google Sheets OAuth2 credentials.

10. **Verify Credentials:**  
    - Ensure all Google Sheets nodes use the same Google OAuth2 credentials with permissions to read, write, create, and delete sheets in the relevant spreadsheets.  
    - Update the HTTP Request node with proper API header authentication if necessary.  

11. **Replace Placeholder Spreadsheet IDs:**  
    - Replace `YOUR_MASTER_SPREADSHEET_ID` in HTTP Request and Read Spreadsheet1 nodes with the actual ID of the master spreadsheet you want to copy from.  

12. **Test Workflow:**  
    - Save and activate workflow.  
    - Manually trigger "When clicking 'Test workflow'".  
    - Monitor execution, verify new spreadsheet creation, sheets duplication, and data accuracy.  

---

### 5. General Notes & Resources

| Note Content                                                                                                                   | Context or Link                                                                                      |
|-------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| Replace spreadsheet ID placeholders in HTTP Request and Read Spreadsheet nodes before running the workflow.                    | Important operational instruction                                                                   |
| Credentials must be updated and authorized with appropriate Google Sheets API scopes (read, write, create, delete).            | Credential setup                                                                                     |
| Default "Sheet1" sheet is removed automatically after all sheets are copied to keep the new spreadsheet clean.                 | Workflow cleanup step                                                                               |
| Column structure between master and new sheets must be consistent to avoid data mismatch during the write operation.           | Data integrity requirement                                                                          |
| Workflow uses Google Sheets API v4 and Google Sheets OAuth2 authentication.                                                    | API and authentication requirements                                                                |
| Manual trigger is designed for on-demand duplication; scheduling can be added if needed.                                       | Workflow start method                                                                               |

---

**Disclaimer:** The content provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and public.