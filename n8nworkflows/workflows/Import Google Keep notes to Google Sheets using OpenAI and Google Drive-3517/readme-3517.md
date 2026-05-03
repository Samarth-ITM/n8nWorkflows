Import Google Keep notes to Google Sheets using OpenAI and Google Drive

https://n8nworkflows.xyz/workflows/import-google-keep-notes-to-google-sheets-using-openai-and-google-drive-3517


# Import Google Keep notes to Google Sheets using OpenAI and Google Drive

### 1. Workflow Overview

This n8n workflow automates the import and processing of Google Keep notes exported as JSON files into a structured Google Sheet. It targets users who export their Google Keep notes via Google Takeout and want to transform the unstructured JSON data into a searchable, filterable spreadsheet. The workflow optionally applies AI-powered summarization or extraction using OpenAI to enrich the notes before exporting.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception & File Discovery:** Triggering the workflow manually and searching for Google Keep JSON files in a specified Google Drive folder.

- **1.2 File Filtering & Batch Processing:** Filtering files by JSON extension and processing them in batches to handle large volumes efficiently.

- **1.3 File Download & Content Extraction:** Downloading each JSON file and extracting the note content.

- **1.4 Note Filtering:** Filtering notes based on archival status and optional content criteria.

- **1.5 AI Processing (Optional):** Applying OpenAI models to analyze or summarize note content.

- **1.6 Data Preparation & Export:** Mapping extracted and AI-processed data fields and appending or updating rows in a Google Sheet.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & File Discovery

- **Overview:**  
  This block initiates the workflow manually and searches a specified Google Drive folder for files, targeting the Google Keep export folder.

- **Nodes Involved:**  
  - When clicking ‘Test workflow’ (Manual Trigger)  
  - Search in "Keep" folder (Google Drive)

- **Node Details:**

  - **When clicking ‘Test workflow’**  
    - Type: Manual Trigger  
    - Role: Starts the workflow on manual user action.  
    - Configuration: Default, no parameters.  
    - Inputs: None  
    - Outputs: Triggers the next node.  
    - Edge Cases: None typical; user must trigger manually.

  - **Search in "Keep" folder**  
    - Type: Google Drive (File/Folder Search)  
    - Role: Searches for files inside a specific Google Drive folder by folder ID.  
    - Configuration:  
      - Folder ID set to the Google Keep export folder (example: "1BggjRVCqyDnECK_mB2M-PYareptQv99P").  
      - Limits results to 2 files (likely for testing; can be increased).  
      - Searches for files only (not folders).  
    - Inputs: Trigger from manual node.  
    - Outputs: List of files found in the folder.  
    - Credentials: Google Drive OAuth2 required.  
    - Edge Cases:  
      - Folder ID must be correct and accessible.  
      - Permissions errors if OAuth token lacks access.  
      - Empty folder returns no files, stopping workflow early.

---

#### 2.2 File Filtering & Batch Processing

- **Overview:**  
  Filters the discovered files to only process JSON files and splits them into batches of 10 for manageable processing.

- **Nodes Involved:**  
  - Loop every 10 items (SplitInBatches)  
  - If extension is json (Filter)

- **Node Details:**

  - **Loop every 10 items**  
    - Type: SplitInBatches  
    - Role: Processes files in batches of 10 to avoid overloading downstream nodes.  
    - Configuration: Batch size set to 10.  
    - Inputs: Files from Google Drive search.  
    - Outputs: Batches of files.  
    - Edge Cases:  
      - If fewer than 10 files, processes all at once.  
      - Large file sets handled gracefully.

  - **If extension is json**  
    - Type: Filter (If)  
    - Role: Filters files to only those with `.json` extension.  
    - Configuration: Checks if file name ends with `.json` (case sensitive).  
    - Inputs: Batch of files.  
    - Outputs: Files passing the filter.  
    - Edge Cases:  
      - Files without `.json` extension are discarded.  
      - Case sensitivity may exclude `.JSON` files unless adjusted.

---

#### 2.3 File Download & Content Extraction

- **Overview:**  
  Downloads each filtered JSON file and extracts the note content from the JSON structure.

- **Nodes Involved:**  
  - Download the files (Google Drive)  
  - Extract from File (ExtractFromFile)

- **Node Details:**

  - **Download the files**  
    - Type: Google Drive (Download)  
    - Role: Downloads the actual JSON file content for processing.  
    - Configuration: Uses file ID from previous node to download.  
    - Inputs: Filtered JSON files.  
    - Outputs: File binary data.  
    - Credentials: Google Drive OAuth2 required.  
    - Edge Cases:  
      - Download failures due to permissions or network issues.  
      - Large files may cause timeouts.

  - **Extract from File**  
    - Type: ExtractFromFile  
    - Role: Parses the downloaded JSON file to extract note data.  
    - Configuration: Operation set to parse JSON content.  
    - Inputs: Binary file data from download node.  
    - Outputs: JSON objects representing individual notes.  
    - Edge Cases:  
      - Malformed JSON files cause parsing errors.  
      - Empty files yield no data.

---

#### 2.4 Note Filtering

- **Overview:**  
  Filters extracted notes to exclude archived notes and optionally filter by content keywords.

- **Nodes Involved:**  
  - Filter (If)  
  - If is archived is false (If)

- **Node Details:**

  - **Filter**  
    - Type: If  
    - Role: Filters notes whose content contains specific keywords ("dépensé" or "depense").  
    - Configuration: Checks if `textContentHtml` contains either keyword (case sensitive).  
    - Inputs: Extracted notes JSON.  
    - Outputs: Notes matching keyword criteria.  
    - Edge Cases:  
      - Notes without `textContentHtml` field may fail condition.  
      - Case sensitivity may exclude some matches.

  - **If is archived is false**  
    - Type: If  
    - Role: Filters notes where `isArchived` is false (i.e., active notes).  
    - Configuration: Boolean check on `data.isArchived` field.  
    - Inputs: Filtered notes from previous node.  
    - Outputs: Non-archived notes.  
    - Edge Cases:  
      - Missing `isArchived` field may cause errors or false negatives.

---

#### 2.5 AI Processing (Optional)

- **Overview:**  
  Applies AI processing to each note to extract or summarize information using OpenAI models.

- **Nodes Involved:**  
  - Put some AI treatment here if you need it (LangChain Agent)  
  - OpenAI Chat Model (LM Chat OpenAI)

- **Node Details:**

  - **Put some AI treatment here if you need it**  
    - Type: LangChain Agent  
    - Role: Sends note content to AI for processing, e.g., extracting amounts in euros.  
    - Configuration:  
      - Prompt template instructs AI to extract euro amounts from note text.  
      - Uses `data.textContent` as input.  
    - Inputs: Non-archived notes.  
    - Outputs: AI-processed output (e.g., extracted amount).  
    - Edge Cases:  
      - AI API errors or rate limits.  
      - Notes without relevant content may yield empty or incorrect AI output.

  - **OpenAI Chat Model**  
    - Type: LM Chat OpenAI  
    - Role: Provides the language model for the LangChain Agent node.  
    - Configuration: Uses GPT-4o-mini model.  
    - Credentials: OpenAI API key required.  
    - Edge Cases:  
      - API key invalid or expired.  
      - Model availability or quota issues.

---

#### 2.6 Data Preparation & Export

- **Overview:**  
  Maps extracted and AI-processed data fields into a structured format and appends or updates rows in a Google Sheet.

- **Nodes Involved:**  
  - Set the fields for export (Set)  
  - Add to google sheet (Google Sheets)

- **Node Details:**

  - **Set the fields for export**  
    - Type: Set  
    - Role: Maps note fields (`textContent`, `Edited`, `Created`, `isArchived`, `Amount`) for export.  
    - Configuration:  
      - Converts timestamps to locale strings.  
      - Uses AI output for `Amount` field.  
    - Inputs: AI-processed notes.  
    - Outputs: Structured data ready for spreadsheet.  
    - Edge Cases:  
      - Missing fields cause empty values.  
      - Date conversion depends on valid timestamps.

  - **Add to google sheet**  
    - Type: Google Sheets  
    - Role: Appends or updates rows in the target Google Sheet.  
    - Configuration:  
      - Auto-maps input data to columns: `textContent`, `Edited`, `Created`, `isArchived`.  
      - Sheet name and document ID set to specific spreadsheet.  
      - Operation: appendOrUpdate.  
    - Inputs: Structured note data.  
    - Credentials: Google Sheets OAuth2 required.  
    - Edge Cases:  
      - Permission errors if OAuth token lacks sheet access.  
      - Data type mismatches or schema changes in sheet.

---

### 3. Summary Table

| Node Name                     | Node Type                          | Functional Role                          | Input Node(s)                 | Output Node(s)                 | Sticky Note                                                                                               |
|-------------------------------|----------------------------------|----------------------------------------|------------------------------|-------------------------------|----------------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’  | Manual Trigger                   | Starts workflow manually                | None                         | Search in "Keep" folder        |                                                                                                          |
| Search in "Keep" folder        | Google Drive                    | Searches for files in Keep folder      | When clicking ‘Test workflow’| Loop every 10 items            | ## Set the directory Where you put the files                                                             |
| Loop every 10 items            | SplitInBatches                  | Processes files in batches of 10       | Search in "Keep" folder       | If extension is json           |                                                                                                          |
| If extension is json           | Filter (If)                    | Filters files by .json extension       | Loop every 10 items           | Download the files             |                                                                                                          |
| Download the files             | Google Drive (Download)         | Downloads JSON files                    | If extension is json          | Extract from File              |                                                                                                          |
| Extract from File              | ExtractFromFile                 | Parses JSON to extract note content    | Download the files            | Filter                        |                                                                                                          |
| Filter                        | Filter (If)                    | Filters notes by keyword content       | Extract from File             | If is archived is false        | ## Filter the files If you need the content to contain a word, or after a certain date. If you don't need to filter it, just remove the node |
| If is archived is false        | Filter (If)                    | Filters out archived notes              | Filter                       | Put some AI treatment here if you need it |                                                                                                          |
| Put some AI treatment here if you need it | LangChain Agent               | Applies AI processing to notes         | If is archived is false       | Set the fields for export      | ## Process each file with AI If you need the extract some information from the context, you can do it here. If you don't need it, just delete the node |
| OpenAI Chat Model              | LM Chat OpenAI                 | Provides AI model for LangChain Agent  | None (linked internally)      | Put some AI treatment here if you need it |                                                                                                          |
| Set the fields for export      | Set                            | Maps note fields for export             | Put some AI treatment here if you need it | Add to google sheet           |                                                                                                          |
| Add to google sheet            | Google Sheets                  | Appends or updates Google Sheet rows   | Set the fields for export     | Loop every 10 items (loop end) | ## Create an empty google sheet file That will get your entries from the notes                            |
| Sticky Note                   | Sticky Note                    | Instructional notes                     | None                         | None                         | ## How to export your Google keep notes * Google has a dedicated service for exporting your google data, called [Google Takeout](https://takeout.google.com/), you'll have to login  it. * Click on "Deselect all" then select only Google Keep and click on "Next". - Select the destination (use "Send download link via mail" as you'll have to uncompress a zip file before to send it again to Google Drive) - Upload to Google Drive all json files from your uncompresed file, to specific directory and you are ready to start! |
| Sticky Note1                  | Sticky Note                    | Instructional note on Google Sheet setup | None                         | None                         | ## Create an empty google sheet file That will get your entries from the notes                            |
| Sticky Note2                  | Sticky Note                    | Instructional note on folder setup     | None                         | None                         | ## Set the directory Where you put the files                                                             |
| Sticky Note3                  | Sticky Note                    | Instructional note on filtering files  | None                         | None                         | ## Filter the files If you need the content to contain a word, or after a certain date. If you don't need to filter it, just remove the node |
| Sticky Note4                  | Sticky Note                    | Instructional note on AI processing     | None                         | None                         | ## Process each file with AI If you need the extract some information from the context, you can do it here. If you don't need it, just delete the node |
| Sticky Note5                  | Sticky Note                    | Setup instructions                      | None                         | None                         | ## Setup * Export your Google Keep notes (see "how to export your Google Keep notes") - Connect Google Drive, OpenAI, and Google Sheets in n8n. - Set the correct folder path for your notes in the “Search in ‘Keep’ folder” node. - Point the Google Sheet node to your spreadsheet |
| Sticky Note6                  | Sticky Note                    | Contact information                     | None                         | None                         | ## Contact me ### If you need some help with this workflow: Write to me: [thomas@pollup.net](mailto:thomas@pollup.net) |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Type: Manual Trigger  
   - Purpose: To start the workflow manually.

2. **Add Google Drive Node to Search Files**  
   - Type: Google Drive (File/Folder Search)  
   - Configure:  
     - Operation: Search files  
     - Folder ID: Set to your Google Keep export folder ID in Google Drive.  
     - Limit: Set to desired number (e.g., 100 for production).  
   - Credentials: Connect your Google Drive OAuth2 credentials.  
   - Connect output from Manual Trigger.

3. **Add SplitInBatches Node**  
   - Type: SplitInBatches  
   - Configure: Batch size = 10  
   - Connect input from Google Drive search node.

4. **Add Filter Node to Check File Extension**  
   - Type: If (Filter)  
   - Condition: Check if file name ends with `.json` (case sensitive).  
   - Connect input from SplitInBatches node.

5. **Add Google Drive Node to Download Files**  
   - Type: Google Drive (Download)  
   - Configure: Use file ID from filtered files.  
   - Credentials: Use same Google Drive OAuth2 credentials.  
   - Connect input from Filter node.

6. **Add ExtractFromFile Node**  
   - Type: ExtractFromFile  
   - Operation: fromJson  
   - Connect input from Download node.

7. **Add Filter Node to Filter Notes by Content Keywords**  
   - Type: If  
   - Condition: Check if `data.textContentHtml` contains "dépensé" OR "depense" (adjust keywords as needed).  
   - Connect input from ExtractFromFile node.

8. **Add Filter Node to Exclude Archived Notes**  
   - Type: If  
   - Condition: `data.isArchived` equals false (boolean).  
   - Connect input from previous Filter node.

9. **Add LangChain Agent Node for AI Processing (Optional)**  
   - Type: LangChain Agent  
   - Configure:  
     - Prompt: "Extract the amount in euros of the input. Output just the amount and nothing else. Here is the input: {{ $json.data.textContent }}"  
     - Prompt type: Define  
   - Credentials: Connect OpenAI API credentials.  
   - Connect input from archived notes filter node.

10. **Add OpenAI Chat Model Node**  
    - Type: LM Chat OpenAI  
    - Configure: Model = GPT-4o-mini (or preferred model)  
    - Credentials: OpenAI API key  
    - Connect as AI model for LangChain Agent node.

11. **Add Set Node to Map Fields for Export**  
    - Type: Set  
    - Configure fields:  
      - `textContent`: `={{ $('If is archived is false').item.json.data.textContent }}`  
      - `Edited`: Convert `userEditedTimestampUsec` to locale string  
      - `Created`: Convert `createdTimestampUsec` to locale string  
      - `isArchived`: `={{ $('If is archived is false').item.json.data.isArchived }}`  
      - `Amount`: AI output from LangChain Agent node  
    - Connect input from LangChain Agent node.

12. **Add Google Sheets Node to Append or Update Rows**  
    - Type: Google Sheets  
    - Configure:  
      - Operation: appendOrUpdate  
      - Document ID: Your target Google Sheet ID  
      - Sheet Name: Target sheet (e.g., "Sheet1")  
      - Columns: Map fields (`textContent`, `Edited`, `Created`, `isArchived`, `Amount`)  
    - Credentials: Connect Google Sheets OAuth2 credentials.  
    - Connect input from Set node.

13. **Connect Google Sheets Node output to SplitInBatches Node input**  
    - This closes the batch processing loop.

14. **Add Sticky Notes for Documentation**  
    - Add sticky notes with instructions on exporting Google Keep notes, setting folder paths, filtering, AI usage, and Google Sheet setup as per original workflow.

---

### 5. General Notes & Resources

| Note Content                                                                                                           | Context or Link                                                                                 |
|------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| How to export your Google Keep notes: Use [Google Takeout](https://takeout.google.com/), select only Google Keep, and download your data. Upload the extracted JSON files to a Google Drive folder. | Sticky Note on workflow start.                                                                 |
| Setup instructions: Connect Google Drive, OpenAI, and Google Sheets credentials in n8n. Set folder path and spreadsheet ID accordingly. | Sticky Note near workflow start.                                                                |
| Filtering tips: Customize the Filter node to extract notes by keywords or dates as needed.                             | Sticky Note near Filter node.                                                                   |
| AI processing: Use the LangChain Agent node to apply custom AI prompts for summarization or data extraction.           | Sticky Note near AI processing nodes.                                                          |
| Contact for help: Email thomas@pollup.net for assistance with this workflow.                                           | Sticky Note at bottom left of workflow.                                                        |

---

This documentation provides a comprehensive understanding of the workflow, enabling users and AI agents to reproduce, modify, and troubleshoot the process of importing Google Keep notes into Google Sheets with optional AI enrichment.