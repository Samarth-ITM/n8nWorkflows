Create Emotional Stories with Gemini AI: Generate Images and Veo3 JSON Prompts

https://n8nworkflows.xyz/workflows/create-emotional-stories-with-gemini-ai--generate-images-and-veo3-json-prompts-8158


# Create Emotional Stories with Gemini AI: Generate Images and Veo3 JSON Prompts

### 1. Workflow Overview

This workflow automates the creation of emotional stories featuring a specific character, generates corresponding video prompts in Veo 3 JSON format, and creates AI-generated images using Google Gemini AI. It organizes assets and data within Google Drive and Google Sheets, maintaining a full lifecycle from story generation to image creation and file management.

The workflow is logically divided into these blocks:

- **1.1 Input Reception and Initialization**: Manual trigger starts the workflow; AI suggests a unique folder name.
- **1.2 Folder and Sheet Setup**: Creates a Google Drive folder and a Google Sheet to store story scenes and metadata.
- **1.3 Emotional Story Generation**: Uses AI to generate a 5-scene emotional story with defined character attributes, parses and structures the output.
- **1.4 Sheet Data Preparation**: Prepares individual story scene rows and appends them to the Google Sheet.
- **1.5 Image and Veo 3 JSON Prompt Generation**: For each scene, converts prompts into detailed Veo 3 JSON format and generates AI images via Google Gemini.
- **1.6 File Upload and Management**: Uploads images to Google Drive, moves files into the created folder, and updates Google Sheets with status and URLs.
- **1.7 Status Checking and Looping**: Periodically checks the status of story scenes to process pending items and update statuses accordingly.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and Initialization

- **Overview**: Initiates workflow execution manually and triggers AI to suggest a unique folder name.
- **Nodes Involved**:  
  - When clicking ‘Execute workflow’ (Manual Trigger)  
  - Folder Name (AI LLM Chain)  
  - Json parser1 (Structured Output Parser)  

- **Node Details**:

  - **When clicking ‘Execute workflow’**  
    - Type: Manual Trigger node  
    - Role: Starts workflow manually. No parameters.  
    - Inputs: None  
    - Outputs: Triggers Folder Name node  
    - Edge cases: User forgets to trigger; no automatic start.

  - **Folder Name**  
    - Type: AI Language Model Chain (Google Gemini LLM)  
    - Role: Generates a random folder name with timestamp for story assets.  
    - Configuration: Prompt instructs AI to “Act as folder name suggestion” for emotional story folder name.  
    - Inputs: Trigger from manual node  
    - Outputs: JSON with folder_name field parsed by Json parser1  
    - Edge cases: AI might generate invalid folder names (empty or special chars).  
    - Credentials: Google Palm API key required.

  - **Json parser1**  
    - Type: Structured Output Parser for AI responses  
    - Role: Parses AI-generated folder name JSON output.  
    - Configuration: Expects JSON with {"folder_name": "folder name"}  
    - Inputs: From Folder Name node  
    - Outputs: Parsed JSON for folder creation  
    - Edge cases: Invalid or malformed JSON from AI  
    - Version: Uses Langchain output parser v1.3

---

#### 1.2 Folder and Sheet Setup

- **Overview**: Creates a Google Drive folder using AI-generated name; creates a new Google Sheet to track story scenes.  
- **Nodes Involved**:  
  - Create folder (Google Drive)  
  - Create sheet (Google Sheets)  
  - Edit Fields (Set Node)  
  - Append row in sheet1 (Google Sheets Append)  

- **Node Details**:

  - **Create folder**  
    - Type: Google Drive node (folder resource)  
    - Role: Creates folder in Google Drive using AI-generated folder name.  
    - Configuration: Uses “My Drive” as base, parent folder ID set to a specific folder (“prompt”)  
    - Credentials: Google Drive OAuth2  
    - Inputs: Parsed folder_name from Json parser1  
    - Outputs: Folder metadata including folder ID for subsequent file operations  
    - Edge cases: Permission errors, rate limits, name conflicts.

  - **Create sheet**  
    - Type: Google Sheets node (create operation)  
    - Role: Creates a new sheet within a fixed Google Sheet document for storing story scenes.  
    - Configuration: Title set dynamically to folder name  
    - Credentials: Google Sheets OAuth2  
    - Inputs: Folder name from Create folder output  
    - Outputs: Sheet metadata including sheet ID and spreadsheet ID  
    - Edge cases: API limits, document access errors.

  - **Edit Fields**  
    - Type: Set node  
    - Role: Initializes or clears key column fields for new rows in the sheet.  
    - Configuration: Sets empty values for Scene, Character Prompt, Action Prompt, Status, Updated Prompt, Image Url  
    - Inputs: From Create sheet node  
    - Outputs: Prepared JSON for appending rows  
    - Edge cases: Misconfigured field names can cause data mismatch.

  - **Append row in sheet1**  
    - Type: Google Sheets Append operation  
    - Role: Appends initialized empty rows or metadata in the newly created sheet.  
    - Configuration: Auto maps input data to sheet columns  
    - Credentials: Google Sheets OAuth2  
    - Inputs: From Edit Fields node  
    - Outputs: Confirmation of appended rows  
    - Edge cases: Sheet column mismatch, permission errors.

---

#### 1.3 Emotional Story Generation

- **Overview**: Generates a 5-scene emotional story about a Pakistani girl named Yusra using AI, parses structured output.  
- **Nodes Involved**:  
  - Story Creator (AI LLM Chain)  
  - Json parser (Structured Output Parser)  
  - Code (JavaScript)  

- **Node Details**:

  - **Story Creator**  
    - Type: AI Language Model Chain (Google Gemini)  
    - Role: Generates story with 5 scenes about Yusra in desi dress, aged 20-25.  
    - Configuration: Prompt instructs AI to create emotional story with 5 scenes, character details specified.  
    - Inputs: From Json parser (which is parsing prior AI or empty input)  
    - Outputs: JSON array with objects containing character_prompt and action_prompt for each scene  
    - Credentials: Google Palm API  
    - Edge cases: AI generates incomplete or malformed JSON, excessive length.

  - **Json parser**  
    - Type: Structured Output Parser  
    - Role: Parses AI output into structured array format with scene prompts.  
    - Configuration: Expects array of 5 scene objects each with character_prompt and action_prompt strings.  
    - Inputs: From Story Creator node  
    - Outputs: JSON array for further processing  
    - Edge cases: Parsing failures if AI output is invalid JSON.

  - **Code**  
    - Type: JavaScript Code node  
    - Role: Extracts and flattens the array items from AI response into separate workflow items, each containing one scene’s prompts.  
    - Configuration: Script iterates over AI output array and returns individual JSON objects with character_prompt and action_prompt.  
    - Inputs: From Json parser node  
    - Outputs: One output item per story scene for batch processing  
    - Edge cases: Null or missing keys in AI output; script errors.

---

#### 1.4 Sheet Data Preparation

- **Overview**: Loops over story scenes, appends each scene’s data as a new row in Google Sheet with status "pending".  
- **Nodes Involved**:  
  - Loop Over Items (Split In Batches)  
  - Append row in sheet (Google Sheets Append)  
  - Rows in sheet (Google Sheets Read Rows)  
  - If3 (Conditional node)  
  - Limit (Limit node)  
  - Wait (Wait node)  

- **Node Details**:

  - **Loop Over Items**  
    - Type: Split In Batches  
    - Role: Processes each story scene item individually in batches (default batch size).  
    - Inputs: From Code node (scene items)  
    - Outputs: One item per batch to Append row in sheet  
    - Edge cases: Large batch sizes may cause API limits.

  - **Append row in sheet**  
    - Type: Google Sheets Append  
    - Role: Adds each scene to the Google Sheet with columns Scene, Character Prompt, Action Prompt, and status "pending".  
    - Credentials: Google Sheets OAuth2  
    - Inputs: From Loop Over Items node  
    - Outputs: Confirmation of added rows  
    - Edge cases: API errors, data mapping issues.

  - **Rows in sheet**  
    - Type: Google Sheets Read Rows  
    - Role: Reads all rows from story sheet to check/update statuses.  
    - Credentials: Google Sheets OAuth2  
    - Inputs: From previous update nodes or periodic checks  
    - Outputs: Current rows data for filtering  
    - Edge cases: Pagination limits, empty sheets.

  - **If3**  
    - Type: Conditional node (IF)  
    - Role: Filters rows where Status equals "pending" to process next steps only for pending items.  
    - Inputs: Rows from sheet  
    - Outputs: Routes to Limit node for pending rows, or to get updated rows for others  
    - Edge cases: Status field missing or inconsistent.

  - **Limit**  
    - Type: Limit node  
    - Role: Restricts number of pending items processed per cycle (default limit).  
    - Inputs: Pending rows from If3  
    - Outputs: Limited number of items to Wait node  
    - Edge cases: Too low limits delay processing; too high may cause timeouts.

  - **Wait**  
    - Type: Wait node  
    - Role: Introduces delay (1 minute) before processing prompt conversion to avoid rate limits or ensure data readiness.  
    - Inputs: Limited pending rows  
    - Outputs: To Prompt converter1 node  
    - Edge cases: Workflow stuck if wait duration too long.

---

#### 1.5 Image and Veo 3 JSON Prompt Generation

- **Overview**: Converts prompts into detailed Veo 3 JSON format, generates images via Google Gemini AI, uploads images to Google Drive, and updates Google Sheets.  
- **Nodes Involved**:  
  - Prompt converter1 (AI LLM Chain)  
  - Json parser2 (Structured Output Parser)  
  - Generate an image (Google Gemini Image Generation)  
  - Upload file (Google Drive Upload)  
  - Update row in sheet (Google Sheets Update)  

- **Node Details**:

  - **Prompt converter1**  
    - Type: AI Language Model Chain (Google Gemini)  
    - Role: Converts natural language prompts (character + action) into structured Veo 3 JSON video generation prompts.  
    - Configuration: Complex system prompt instructs AI to output only valid JSON with detailed video parameters (prompt, negative_prompt, aspect_ratio, camera_movement, style, etc.)  
    - Inputs: From Wait node (pending rows)  
    - Outputs: JSON structured Veo 3 video prompts parsed by Json parser2  
    - Credentials: Google Palm API  
    - Edge cases: AI output invalid JSON, missing parameters.

  - **Json parser2**  
    - Type: Structured Output Parser  
    - Role: Parses the AI-generated Veo 3 JSON prompt for downstream usage.  
    - Inputs: From Prompt converter1  
    - Outputs: Parsed JSON object with video prompt parameters  
    - Edge cases: Parsing failure on malformed JSON.

  - **Generate an image**  
    - Type: Google Gemini Image Generation node  
    - Role: Generates an image based on combined prompt parameters from Veo 3 JSON output.  
    - Configuration: Constructs prompt string by concatenating all relevant Veo 3 JSON fields (prompt, negative_prompt, mood, style, etc.)  
    - Inputs: Parsed JSON from Json parser2  
    - Outputs: Image data with file metadata including image ID  
    - Credentials: Google Palm API  
    - Edge cases: Generation failures, API quota limits.

  - **Upload file**  
    - Type: Google Drive Upload node  
    - Role: Uploads generated image file to Google Drive under the predefined folder.  
    - Configuration: File name prepended with scene info, saved in the folder created earlier.  
    - Inputs: From Generate an image node  
    - Outputs: File metadata with Drive file ID  
    - Credentials: Google Drive OAuth2  
    - Edge cases: Upload failures, file size limits, permission issues.

  - **Update row in sheet**  
    - Type: Google Sheets Update node  
    - Role: Updates the story sheet row with image URL (Drive file ID) and status "created", and adds updated prompt.  
    - Inputs: From Upload file node and row data  
    - Outputs: Confirmation of update  
    - Credentials: Google Sheets OAuth2  
    - Edge cases: Row indexing errors, concurrent update conflicts.

---

#### 1.6 File Upload and Management

- **Overview**: Moves uploaded image files into the created folder in Google Drive and updates sheet status to "moved".  
- **Nodes Involved**:  
  - If (conditional node)  
  - Move file (Google Drive Move)  
  - Update Moved (Google Sheets Update)  
  - get updated rows (Google Sheets Read Rows)  
  - No Operation, do nothing (NoOp node)  

- **Node Details**:

  - **If**  
    - Type: Conditional node  
    - Role: Checks if the sheet row Status is "created" before moving files.  
    - Inputs: Updated rows from sheet  
    - Outputs: Routes to Move file node if status is "created", otherwise to No Operation node.  
    - Edge cases: Status field inconsistency.

  - **Move file**  
    - Type: Google Drive Move operation  
    - Role: Moves the uploaded image file into the newly created folder (using folder ID).  
    - Inputs: File ID from Update row in sheet, Folder ID from Create folder  
    - Credentials: Google Drive OAuth2  
    - Outputs: Move confirmation and metadata  
    - Edge cases: Drive permission or file locking issues.

  - **Update Moved**  
    - Type: Google Sheets Update  
    - Role: Updates sheet row Status to "moved" after file is moved in Drive.  
    - Inputs: From Move file node and row data  
    - Credentials: Google Sheets OAuth2  
    - Outputs: Confirmation  
    - Edge cases: Update conflicts.

  - **get updated rows**  
    - Type: Google Sheets Read Rows  
    - Role: Reads updated sheet rows for status checks and next processing cycle.  
    - Credentials: Google Sheets OAuth2  
    - Edge cases: Pagination, empty sheets.

  - **No Operation, do nothing**  
    - Type: NoOp node  
    - Role: Placeholder for branch where no action is needed.  
    - Inputs: From If node when status is not "created"  
    - Outputs: None

---

#### 1.7 Status Checking and Looping

- **Overview**: Reads updated Google Sheets rows, filters by status, limits batch size, and waits before next prompt conversion and image generation cycle to handle pending scenes.  
- **Nodes Involved**:  
  - Rows in sheet (Google Sheets Read Rows)  
  - If3 (Conditional node)  
  - Limit (Limit node)  
  - Wait (Wait node)  

- **Node Details**:

  - **Rows in sheet**  
    - Reads all rows for status evaluation  
    - Inputs: From Update nodes or periodic triggers  
    - Outputs: Row data for filtering

  - **If3**  
    - Filters rows with Status == "pending"  

  - **Limit**  
    - Limits number of rows processed each cycle  

  - **Wait**  
    - Waits 1 minute before processing to avoid API limit issues  

- The looping mechanism ensures that any new or pending story scenes are processed sequentially without overwhelming APIs.

---

### 3. Summary Table

| Node Name              | Node Type                         | Functional Role                               | Input Node(s)              | Output Node(s)                   | Sticky Note                                                                                              |
|------------------------|----------------------------------|-----------------------------------------------|----------------------------|---------------------------------|--------------------------------------------------------------------------------------------------------|
| Sticky Note7           | Sticky Note                      | Author info and contact details               | None                       | None                            | Muhammad Farooq Iqbal - Automation Expert & n8n Creator; contact info and portfolio link                |
| When clicking ‘Execute workflow’ | Manual Trigger                 | Workflow start trigger                        | None                       | Folder Name                     |                                                                                                        |
| Folder Name            | AI LLM Chain (Google Gemini)      | Generate random folder name                    | When clicking ‘Execute workflow’ | Create folder                  |                                                                                                        |
| Json parser1           | Langchain Structured Output Parser| Parse folder name JSON                         | Folder Name                | Create folder                   |                                                                                                        |
| Create folder          | Google Drive (Folder Create)      | Create Google Drive folder                      | Json parser1               | Create sheet                   |                                                                                                        |
| Create sheet           | Google Sheets (Create Sheet)      | Create new Google Sheet for story scenes       | Create folder              | Edit Fields                   |                                                                                                        |
| Edit Fields            | Set node                         | Initialize empty row fields                     | Create sheet               | Append row in sheet1            |                                                                                                        |
| Append row in sheet1   | Google Sheets (Append)             | Append empty rows or metadata                   | Edit Fields                | Story Creator                 |                                                                                                        |
| Story Creator          | AI LLM Chain (Google Gemini)      | Generate 5-scene emotional story JSON          | Append row in sheet1 or Json parser | Json parser                  |                                                                                                        |
| Json parser            | Langchain Structured Output Parser| Parse story scenes JSON array                   | Story Creator              | Code                         |                                                                                                        |
| Code                   | JavaScript Code                   | Flatten story scenes array to individual items | Json parser                | Loop Over Items                |                                                                                                        |
| Loop Over Items        | Split In Batches                  | Process each story scene separately             | Code                       | Append row in sheet            |                                                                                                        |
| Append row in sheet    | Google Sheets (Append)             | Append each story scene as row with status pending | Loop Over Items            | Rows in sheet                 |                                                                                                        |
| Rows in sheet          | Google Sheets (Read Rows)          | Read all rows to check processing status       | Append row in sheet or Update row in sheet | If3                         |                                                                                                        |
| If3                    | IF node                          | Filter rows where Status == "pending"           | Rows in sheet              | Limit / get updated rows       |                                                                                                        |
| Limit                  | Limit node                      | Limit number of pending items processed         | If3                        | Wait                         |                                                                                                        |
| Wait                   | Wait node                       | Pause before processing to avoid rate limits    | Limit                      | Prompt converter1             |                                                                                                        |
| Prompt converter1      | AI LLM Chain (Google Gemini)      | Convert prompts to Veo 3 JSON format             | Wait                       | Json parser2                  |                                                                                                        |
| Json parser2           | Langchain Structured Output Parser| Parse Veo 3 JSON prompt                           | Prompt converter1          | Generate an image             |                                                                                                        |
| Generate an image      | Google Gemini Image Generation     | Generate image from prompt                        | Json parser2               | Upload file                   |                                                                                                        |
| Upload file            | Google Drive (Upload)             | Upload generated image to Drive                  | Generate an image          | Update row in sheet           |                                                                                                        |
| Update row in sheet    | Google Sheets (Update Row)         | Update sheet row with image URL and status "created" | Upload file                | Rows in sheet                 |                                                                                                        |
| get updated rows       | Google Sheets (Read Rows)          | Reads updated rows for status processing         | Update Moved or Update row in sheet | If                          |                                                                                                        |
| If                     | IF node                          | Check if row Status == "created" for moving file | get updated rows           | Move file / No Operation       |                                                                                                        |
| Move file              | Google Drive (Move File)           | Move uploaded image to created folder            | If                        | Update Moved                 |                                                                                                        |
| Update Moved           | Google Sheets (Update Row)         | Update sheet Status to "moved" after file move   | Move file                  | get updated rows             |                                                                                                        |
| No Operation, do nothing | NoOp node                      | Placeholder for no action branch                   | If                        | None                        |                                                                                                        |
| Sticky Note            | Sticky Note                      | Section title: Create Folder and File             | None                       | None                         | ## Create Folder and File                                                                               |
| Sticky Note3           | Sticky Note                      | Section title: Create Story & Add in Sheet        | None                       | None                         | ## Create Story & Add in Sheet                                                                          |
| Sticky Note4           | Sticky Note                      | Section title: Create Veo 3 Json Prompts & Images | None                       | None                         | ## Create Veo 3 Json Prompts & Images via Gemini                                                       |
| Sticky Note5           | Sticky Note                      | Section title: Move Files                           | None                       | None                         | ## Move Files                                                                                           |
| Sticky Note1           | Sticky Note                      | Setup instructions summary                         | None                       | None                         | Setup instructions and credentials requirements                                                        |
| chat model             | AI Chat Model (Google Gemini)     | (Not directly connected, possibly legacy or test) | None                       | Folder Name                  |                                                                                                        |
| chat model1            | AI Chat Model (Google Gemini)     | (Not directly connected, possibly legacy or test) | None                       | Story Creator                |                                                                                                        |
| chat model2            | AI Chat Model (Google Gemini)     | (Not directly connected, possibly legacy or test) | None                       | Prompt converter1            |                                                                                                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Name: `When clicking ‘Execute workflow’`  
   - Purpose: Manually start workflow.

2. **Add AI Language Model Node for Folder Name**  
   - Type: Chain LLM using Google Gemini (model: `models/gemini-2.5-flash-lite`)  
   - Prompt: "Act as a folder name suggestion. Suggest a folder name for an emotional story. Always give a random name with timestamp."  
   - Output: Enable structured output parsing expecting JSON with `folder_name`  
   - Connect: Manual Trigger → Folder Name

3. **Add Output Parser Node (Json parser1)**  
   - Use Langchain Structured Output Parser  
   - JSON Schema Example: `{ "folder_name": "folder name" }`  
   - Connect: Folder Name → Json parser1

4. **Create Google Drive Folder Node**  
   - Operation: Create folder  
   - Drive: My Drive  
   - Parent Folder ID: Set to your root or specific folder ID  
   - Folder Name: Expression from Json parser1 output: `{{$json["folder_name"]}}`  
   - Credentials: Google Drive OAuth2  
   - Connect: Json parser1 → Create folder

5. **Create Google Sheets Node to create a new Sheet**  
   - Operation: Create sheet  
   - Document ID: Your master Google Sheet document ID  
   - Title: Use folder name from Create folder node  
   - Credentials: Google Sheets OAuth2  
   - Connect: Create folder → Create sheet

6. **Add Set Node named "Edit Fields"**  
   - Initialize empty fields: Scene, Character Prompt, Action Prompt, Status, Updated Prompt, Image Url (all empty strings)  
   - Connect: Create sheet → Edit Fields

7. **Append Row in Sheet (Append row in sheet1)**  
   - Operation: Append row  
   - Sheet Name: Use sheet ID from Create sheet node  
   - Map empty fields from "Edit Fields" node  
   - Credentials: Google Sheets OAuth2  
   - Connect: Edit Fields → Append row in sheet1

8. **Add AI Language Model Node (Story Creator)**  
   - Model: Google Gemini (same as above)  
   - Prompt: Create a 5-scene emotional story featuring a Pakistani girl aged 20-25 named Yusra in desi dress.  
   - Enable structured output parsing expecting an array of objects with `character_prompt` and `action_prompt`  
   - Connect: Append row in sheet1 → Story Creator

9. **Add Output Parser Node (Json parser)**  
   - Schema: Array of 5 objects with `character_prompt` and `action_prompt`  
   - Connect: Story Creator → Json parser

10. **Add Code Node**  
    - JavaScript to split the array into individual items for batching:  
      ```js
      const outputs = $input.first().json.output;
      const out = [];
      for (const o of outputs) {
        out.push({ json: { scene: o } });
      }
      return out;
      ```  
    - Connect: Json parser → Code

11. **Add SplitInBatches Node (Loop Over Items)**  
    - Batch size default (e.g., 1)  
    - Connect: Code → Loop Over Items

12. **Append Row in Sheet (Append row in sheet)**  
    - Operation: Append  
    - Sheet Name: From Create sheet output  
    - Map Scene, Character Prompt, Action Prompt from current item  
    - Set Status to "pending"  
    - Credentials: Google Sheets OAuth2  
    - Connect: Loop Over Items → Append row in sheet

13. **Read Rows from Sheet (Rows in sheet)**  
    - Operation: Read rows  
    - Sheet Name: From Create sheet  
    - Credentials: Google Sheets OAuth2  
    - Connect: Append row in sheet and Update row in sheet (later steps) → Rows in sheet

14. **Add IF Node (If3)**  
    - Condition: `Status` equals `"pending"`  
    - Connect: Rows in sheet → If3

15. **Add Limit Node**  
    - Limit number of items processed per cycle (default 1 or configurable)  
    - Connect: If3 (true branch) → Limit

16. **Add Wait Node**  
    - Wait 1 minute to manage API rate limits  
    - Connect: Limit → Wait

17. **Add AI Language Model Node (Prompt converter1)**  
    - Model: Google Gemini  
    - System prompt: Convert natural language prompts to Veo 3 JSON video generation format with detailed parameters (copy detailed system prompt from original)  
    - Input: Concatenate Character Prompt and Action Prompt fields  
    - Enable structured output parsing expecting Veo 3 JSON schema  
    - Connect: Wait → Prompt converter1

18. **Add Output Parser Node (Json parser2)**  
    - Parse Veo 3 JSON prompt from AI output  
    - Connect: Prompt converter1 → Json parser2

19. **Add Google Gemini Image Generation Node (Generate an image)**  
    - Use model: `models/gemini-2.0-flash-exp-image-generation`  
    - Construct prompt by concatenating all Veo 3 JSON fields (prompt, negative_prompt, style, mood, etc.) from Json parser2 output  
    - Credentials: Google Palm API  
    - Connect: Json parser2 → Generate an image

20. **Add Google Drive Upload Node (Upload file)**  
    - Upload generated image to Drive folder created earlier  
    - File name: `s_{Scene}.{fileExtension or png}`  
    - Credentials: Google Drive OAuth2  
    - Connect: Generate an image → Upload file

21. **Add Google Sheets Update Node (Update row in sheet)**  
    - Update the corresponding row with Image Url (file ID), Status = "created", Updated Prompt (from Prompt converter1 output)  
    - Credentials: Google Sheets OAuth2  
    - Connect: Upload file → Update row in sheet

22. **Add Google Sheets Read Rows Node (get updated rows)**  
    - Reads updated rows for status check  
    - Connect: Update Moved and Update row in sheet → get updated rows

23. **Add IF Node (If)**  
    - Condition: Status equals "created"  
    - Connect: get updated rows → If

24. **Add Google Drive Move Node (Move file)**  
    - Moves image file into the created folder  
    - Credentials: Google Drive OAuth2  
    - Connect: If (true) → Move file

25. **Add Google Sheets Update Node (Update Moved)**  
    - Updates status to "moved" after file move  
    - Credentials: Google Sheets OAuth2  
    - Connect: Move file → Update Moved

26. **Add No Operation Node (No Operation, do nothing)**  
    - Placeholder for false branch of If  
    - Connect: If (false) → No Operation

27. **Connect Update Moved → get updated rows** (loop for continuous processing).

28. **Add Sticky Notes as Section Headers and Setup Instructions**  
    - Add relevant sticky notes describing blocks and setup guidance.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                  | Context or Link                                                                                     |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| Muhammad Farooq Iqbal - Automation Expert & n8n Creator with 500+ downloads, specializing in AI and workflow automation. Contact: mfarooqiqbal143@gmail.com, +923036991118, LinkedIn: https://linkedin.com/in/muhammadfarooqiqbal, Portfolio: https://mfarooqone.github.io/n8n/ | Author and support contact info                                                                    |
| Setup instructions require configuring Google Drive OAuth2, Google Sheets OAuth2, and Google Gemini API credentials. Update Google Sheets document ID and Drive folder ID accordingly.                                                                           | Setup instructions sticky note                                                                     |
| Veo 3 JSON prompt conversion system prompt includes detailed guidelines and defaults for video generation parameters to ensure consistent and high-quality AI prompt outputs.                                                                                   | Prompt converter1 node system prompt                                                               |
| Workflow depends on Google Gemini AI models for text and image generation: LLM for story and prompt generation, and image generation model `gemini-2.0-flash-exp-image-generation`.                                                                                | AI integration details                                                                             |
| Workflow uses batching and wait nodes to handle API rate limits and ensure smooth processing of multiple story scenes.                                                                                                                                          | Performance and rate limit management                                                              |

---

**Disclaimer:** The provided text is exclusively derived from an automated n8n workflow. It complies fully with content policies and contains no illegal, offensive, or protected elements. All data processed is legal and public.