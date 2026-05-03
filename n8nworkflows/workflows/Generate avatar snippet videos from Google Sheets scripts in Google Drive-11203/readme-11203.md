Generate avatar snippet videos from Google Sheets scripts in Google Drive

https://n8nworkflows.xyz/workflows/generate-avatar-snippet-videos-from-google-sheets-scripts-in-google-drive-11203


# Generate avatar snippet videos from Google Sheets scripts in Google Drive

### 1. Workflow Overview

This workflow automates the generation of avatar video snippets based on script data stored in a Google Sheets document and avatar descriptions from another sheet within the same document. It leverages Google Gemini's Veo model for video generation, uploads the resulting videos to a specified Google Drive folder, and writes back shareable video links into the corresponding rows of the script sheet.

The workflow is structured into these main logical blocks:

- **1.1 Input Trigger and Data Retrieval**: Manual trigger initiates the workflow, which then reads avatar descriptions and script lines from Google Sheets.
- **1.2 Global Variable Setup and Loop Initialization**: Sets key variables and starts looping over each script row.
- **1.3 Video Generation Loop**: For each script entry, prepares inputs and uses the Gemini Veo model to generate a video snippet.
- **1.4 Upload and Link Update**: Uploads the generated video to Google Drive and updates the script sheet row with the shareable link.
- **1.5 Loop Control**: Manages iteration over all script entries until completion.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Trigger and Data Retrieval

- **Overview:**  
This block starts the workflow manually and retrieves the necessary input data: avatar descriptions and script lines from two different sheets within the same Google Sheets document.

- **Nodes Involved:**  
  - When clicking ‘Execute workflow’ (Manual Trigger)  
  - Get Avatar Description (Google Sheets)  
  - Get Script (Google Sheets)

- **Node Details:**  

  - **When clicking ‘Execute workflow’**  
    - Type: Manual Trigger  
    - Role: Entry point for manual workflow execution.  
    - Configuration: No parameters; activates workflow on user command.  
    - Inputs: None  
    - Outputs: Triggers "Get Avatar Description" node.  
    - Edge cases: Workflow will not start unless executed manually.

  - **Get Avatar Description**  
    - Type: Google Sheets (Read)  
    - Role: Fetches the avatar description from a specific sheet (Sheet ID 717424848) within the Google Sheets document.  
    - Configuration:  
      - Document ID: Fixed Google Sheets ID  
      - Sheet ID: 717424848 (named "Gaia" in UI)  
      - Credentials: Google Sheets OAuth2  
    - Inputs: Triggered by manual trigger node.  
    - Outputs: Passes data to "Global Variables" node.  
    - Edge cases: Authentication failure, sheet access denied, empty or malformed data.

  - **Get Script**  
    - Type: Google Sheets (Read)  
    - Role: Retrieves the script rows from a sheet named by the variable "page" (default "Draft 5") in the same Google Sheets document.  
    - Configuration:  
      - Document ID: Same as above  
      - Sheet Name: Set dynamically from variable "page"  
      - Credentials: Google Sheets OAuth2  
    - Inputs: Triggered after "Global Variables" node.  
    - Outputs: Feeds data into the "Loop Over Items" node.  
    - Edge cases: Sheet name variable not set or invalid, access issues.

---

#### 2.2 Global Variable Setup and Loop Initialization

- **Overview:**  
Sets global variables including the page name and description, then initializes batch processing for iterating over each script entry.

- **Nodes Involved:**  
  - Global Variables (Set)  
  - Loop Over Items (SplitInBatches)  
  - Sticky Note1 (Documentation)  
  - Sticky Note2 (Documentation)

- **Node Details:**  

  - **Global Variables**  
    - Type: Set  
    - Role: Assigns fixed variables for the workflow such as the page name ("Draft 5") and dynamically sets Description from the avatar sheet data.  
    - Configuration:  
      - page = "Draft 5" (string)  
      - Description = expression extracting Description field from incoming JSON  
    - Inputs: From "Get Avatar Description" node  
    - Outputs: To "Get Script" node  
    - Edge cases: Expression failure if Description field missing.

  - **Loop Over Items**  
    - Type: SplitInBatches  
    - Role: Controls batch processing to iterate over each script row individually.  
    - Configuration: Default batch size (single processing per item)  
    - Inputs: From "Get Script" node  
    - Outputs: Iterates to "Set Loop Inputs" node  
    - Edge cases: Empty input data (no rows), batch processing errors.

  - **Sticky Note1 and Sticky Note2**  
    - Type: Sticky Note  
    - Role: Provide user documentation and context about the page variable and avatar description usage.  
    - Inputs/Outputs: None (purely informative)  
    - Content Summary:  
      - Sticky Note1 explains defining and retrieving the script page.  
      - Sticky Note2 explains the retrieval of avatar description.

---

#### 2.3 Video Generation Loop

- **Overview:**  
For each script row, this block prepares necessary input fields and calls Google Gemini's Veo model to generate an avatar video snippet based on avatar description, framing, and speech text.

- **Nodes Involved:**  
  - Set Loop Inputs (Set)  
  - Generate a video with Veo (Google Gemini Video Generation)  
  - Sticky Note3 (Documentation)

- **Node Details:**  

  - **Set Loop Inputs**  
    - Type: Set  
    - Role: Combines global avatar description with current script row's speech and framing data to prepare inputs for video generation.  
    - Configuration:  
      - Gaia: Takes Description from "Global Variables" node  
      - Speech: Extracted from current script row's "Speech" field  
      - framing: Extracted from current script row's "Framing" field  
      - Description: Extracted from current script row's "Description" field  
    - Inputs: From "Loop Over Items" node (batch iteration)  
    - Outputs: To "Generate a video with Veo" node  
    - Edge cases: Missing speech/framing/description fields in current row.

  - **Generate a video with Veo**  
    - Type: Google Gemini Video Generation (Langchain node)  
    - Role: Calls Gemini Veo model to generate an 8-second avatar video snippet with specified prompts.  
    - Configuration:  
      - Model ID: "models/veo-3.0-fast-generate-001"  
      - Prompt: Multi-line template including avatar description, framing, and speech text.  
      - Options: Aspect ratio 16:9, duration 8 seconds, allow all person generation, output binary data.  
      - Credentials: Google Palm API OAuth2  
    - Inputs: From "Set Loop Inputs"  
    - Outputs: Binary video data to "Upload video file" node  
    - Edge cases: API authentication errors, generation failures, prompt formatting issues, timeouts.

  - **Sticky Note3**  
    - Type: Sticky Note  
    - Role: Detailed explanation of the video generation loop, including how video snippets are created, uploaded, and linked back.  
    - Content includes Gemini logo and descriptive text about the core automation flow.

---

#### 2.4 Upload and Link Update

- **Overview:**  
Uploads the generated video file to a designated Google Drive folder and updates the corresponding script row with the video’s shareable link.

- **Nodes Involved:**  
  - Upload video file (Google Drive)  
  - Update row in sheet with link to video (Google Sheets)  
  - Move on to next snippet (NoOp)

- **Node Details:**  

  - **Upload video file**  
    - Type: Google Drive (Upload)  
    - Role: Uploads the generated video binary as an MP4 file to a specific Google Drive folder.  
    - Configuration:  
      - File name: "Video {row_number}.mp4" dynamically generated from current loop item row number  
      - Drive ID: "My Drive"  
      - Folder ID: Fixed Google Drive folder ID "14TmRVP_anhGNV6RfCUWL6fLYm0aMwaXt"  
      - Credentials: Google Drive OAuth2  
    - Inputs: From "Generate a video with Veo" node (binary video data)  
    - Outputs: File metadata including webViewLink to "Update row in sheet with link to video" node  
    - Edge cases: Upload failures, permission errors, file naming conflicts.

  - **Update row in sheet with link to video**  
    - Type: Google Sheets (Update)  
    - Role: Updates the original script sheet row with the shareable Google Drive link for the uploaded video.  
    - Configuration:  
      - Document ID and Sheet Name: Same as script sheet ("Draft 5")  
      - Matching column: "row_number" to identify correct row  
      - Columns updated: "Link" field set to uploaded file's webViewLink  
      - Credentials: Google Sheets OAuth2  
    - Inputs: From "Upload video file" node  
    - Outputs: To "Move on to next snippet" node  
    - Edge cases: Row matching failure, permissions, update conflicts.

  - **Move on to next snippet**  
    - Type: NoOp  
    - Role: Acts as a placeholder/connector node to continue the loop iteration by triggering "Loop Over Items" for next batch.  
    - Configuration: None  
    - Inputs: From "Update row in sheet with link to video"  
    - Outputs: Back to "Loop Over Items"  
    - Edge cases: None; purely logical flow control.

---

### 3. Summary Table

| Node Name                       | Node Type                      | Functional Role                                    | Input Node(s)                      | Output Node(s)                              | Sticky Note                                                     |
|--------------------------------|--------------------------------|---------------------------------------------------|----------------------------------|---------------------------------------------|----------------------------------------------------------------|
| When clicking ‘Execute workflow’| Manual Trigger                 | Starts workflow manually                           | None                             | Get Avatar Description                      | ## Get Avatar description: This section gets the description of the Avatar |
| Get Avatar Description          | Google Sheets                 | Reads avatar description sheet                     | When clicking ‘Execute workflow’ | Global Variables                            | ## Get Avatar description: This section gets the description of the Avatar |
| Global Variables               | Set                           | Sets global variables (page name, description)    | Get Avatar Description            | Get Script                                  | ## Define and Get Script: Define the page and prepare script retrieval |
| Get Script                    | Google Sheets                 | Reads script rows from specified page              | Global Variables                 | Loop Over Items                             | ## Define and Get Script: Define the page and prepare script retrieval |
| Loop Over Items                | SplitInBatches                | Iterates over script rows one by one               | Get Script                      | Set Loop Inputs                            |                                                                |
| Set Loop Inputs               | Set                           | Prepares inputs for video generation per row       | Loop Over Items                 | Generate a video with Veo                    |                                                                |
| Generate a video with Veo     | Google Gemini Video Generation| Generates avatar video snippet using Gemini Veo    | Set Loop Inputs                | Upload video file                          | ![gemini](https://upload.wikimedia.org/wikipedia/commons/thumb/8/8a/Google_Gemini_logo.svg/330px-Google_Gemini_logo.svg.png) <br> ## Generate Snippet Videos: Full video generation loop explanation |
| Upload video file             | Google Drive                  | Uploads generated video to Google Drive folder     | Generate a video with Veo       | Update row in sheet with link to video      |                                                                |
| Update row in sheet with link to video | Google Sheets                 | Updates script sheet row with video link           | Upload video file               | Move on to next snippet                     |                                                                |
| Move on to next snippet        | NoOp                          | Controls loop iteration continuation                | Update row in sheet with link to video | Loop Over Items                      |                                                                |
| Sticky Note1                  | Sticky Note                   | Documentation for script page definition           | None                           | None                                        | ## Define and Get Script: Define the page and prepare script retrieval |
| Sticky Note2                  | Sticky Note                   | Documentation for avatar description retrieval      | None                           | None                                        | ## Get Avatar description: This section gets the description of the Avatar |
| Sticky Note3                  | Sticky Note                   | Documentation for video generation loop             | None                           | None                                        | ![gemini](https://upload.wikimedia.org/wikipedia/commons/thumb/8/8a/Google_Gemini_logo.svg/330px-Google_Gemini_logo.svg.png) <br> ## Generate Snippet Videos: Full video generation loop explanation |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Name: "When clicking ‘Execute workflow’"  
   - Type: Manual Trigger  
   - Purpose: Start workflow execution manually.

2. **Create Google Sheets Node to Get Avatar Description**  
   - Name: "Get Avatar Description"  
   - Type: Google Sheets (Read)  
   - Parameters:  
     - Document ID: `1odA_IPe1h1K2jffAmS8KAow3A_iqOzmOKp261s9qjh8`  
     - Sheet ID: `717424848` (or sheet named "Gaia")  
   - Credentials: Google Sheets OAuth2 (e.g., "Google Sheets Angel Access")  
   - Connect input from Manual Trigger node.

3. **Create Set Node to Define Global Variables**  
   - Name: "Global Variables"  
   - Type: Set  
   - Parameters:  
     - Assign variable `page` = "Draft 5" (string)  
     - Assign variable `Description` = Expression: `={{ $json.Description }}` (extract from avatar description sheet)  
   - Connect input from "Get Avatar Description".

4. **Create Google Sheets Node to Get Script Rows**  
   - Name: "Get Script"  
   - Type: Google Sheets (Read)  
   - Parameters:  
     - Document ID: same as above  
     - Sheet Name: Expression `={{ $json.page }}` from "Global Variables" node  
   - Credentials: Google Sheets OAuth2 (same as above)  
   - Connect input from "Global Variables".

5. **Create SplitInBatches Node to Loop Over Script Rows**  
   - Name: "Loop Over Items"  
   - Type: SplitInBatches  
   - Parameters: Default (batch size 1)  
   - Connect input from "Get Script".

6. **Create Set Node to Prepare Loop Inputs**  
   - Name: "Set Loop Inputs"  
   - Type: Set  
   - Parameters:  
     - Gaia = Expression: `={{ $('Global Variables').item.json.Description }}` (avatar description)  
     - Speech = Expression: `={{ $json.Speech }}` (current row speech)  
     - framing = Expression: `={{ $json.Framing }}` (current row framing)  
     - Description = Expression: `={{ $json.Description }}` (current row description)  
   - Connect input from "Loop Over Items" (second output for batch items).

7. **Create Google Gemini Video Generation Node**  
   - Name: "Generate a video with Veo"  
   - Type: Google Gemini (Langchain)  
   - Parameters:  
     - Model ID: `models/veo-3.0-fast-generate-001`  
     - Prompt:  
       ```
       Video opens with a midsize shot of avatar described below. They have a brown background behind them with subtle lightning bolts slowly appearing and disapearing in the background, but it's blurred to not draw attention. 

       avatar:
       {{ $json.Description }}

       Framing:
       {{ $json.framing }}

       She says the following:

       {{ $json.Speech }}
       ```  
     - Options:  
       - Aspect Ratio: 16:9  
       - Duration: 8 seconds  
       - Sample count: 1  
       - Person generation: allow_all  
       - Binary output property: data  
   - Credentials: Google Palm API OAuth2 (e.g., "Angel Gemini Creds")  
   - Connect input from "Set Loop Inputs".

8. **Create Google Drive Node to Upload Video**  
   - Name: "Upload video file"  
   - Type: Google Drive (Upload)  
   - Parameters:  
     - File Name: `=Video {{ $('Loop Over Items').item.json.row_number }}.mp4`  
     - Drive ID: "My Drive"  
     - Folder ID: `14TmRVP_anhGNV6RfCUWL6fLYm0aMwaXt` (target folder)  
   - Credentials: Google Drive OAuth2 (e.g., "Angel Gdrive")  
   - Connect input from "Generate a video with Veo".

9. **Create Google Sheets Node to Update Script Row**  
   - Name: "Update row in sheet with link to video"  
   - Type: Google Sheets (Update)  
   - Parameters:  
     - Document ID and Sheet Name: Same as "Get Script" (page "Draft 5")  
     - Matching column: "row_number"  
     - Columns to update: Set "Link" = `={{ $json.webViewLink }}` (Google Drive shareable link)  
   - Credentials: Google Sheets OAuth2  
   - Connect input from "Upload video file".

10. **Create No Operation Node for Loop Control**  
    - Name: "Move on to next snippet"  
    - Type: NoOp  
    - Connect input from "Update row in sheet with link to video".  
    - Connect output back to "Loop Over Items" to continue batch iteration.

11. **Add Sticky Notes (Optional for Documentation)**  
    - Place sticky notes near relevant nodes for:  
      - Explaining avatar description retrieval  
      - Defining and getting script page  
      - Describing video generation loop and Gemini API usage  

---

### 5. General Notes & Resources

| Note Content                                                                                                                               | Context or Link                                                                                      |
|--------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| This workflow uses Google Gemini Veo model for avatar video generation, requiring Google Palm API credentials.                             | Google Gemini official docs: https://developers.generativeai.google/models/gemini                 |
| Google Sheets and Drive OAuth2 credentials must have appropriate read/write permissions for the target spreadsheet and Drive folder.     | Refer to Google Cloud Console and n8n credential setup guidelines.                                  |
| Video generation prompt includes dynamic insertion of avatar description, framing, and speech text for personalized snippet creation.     | See Sticky Note3 content inside the workflow for detailed prompt and flow description.             |
| Folder ID and Sheet IDs are hardcoded and must be accessible; adjust them if deploying in different environments or accounts.             | Folder link: https://drive.google.com/drive/folders/14TmRVP_anhGNV6RfCUWL6fLYm0aMwaXt                |
| The workflow is designed for manual triggering but can be adapted for scheduled triggers or webhook activation for automation.            |                                                                                                    |

---

**Disclaimer:** The provided content is extracted and analyzed exclusively from an n8n workflow automation. It strictly adheres to content policies and handles only publicly accessible, authorized data.