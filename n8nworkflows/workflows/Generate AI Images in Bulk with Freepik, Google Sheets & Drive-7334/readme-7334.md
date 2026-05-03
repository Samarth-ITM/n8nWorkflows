Generate AI Images in Bulk with Freepik, Google Sheets & Drive

https://n8nworkflows.xyz/workflows/generate-ai-images-in-bulk-with-freepik--google-sheets---drive-7334


# Generate AI Images in Bulk with Freepik, Google Sheets & Drive

### 1. Workflow Overview

This workflow automates the bulk generation of AI images using Freepik’s text-to-image API, leveraging prompts stored in Google Sheets, and then uploads the generated images to a specified Google Drive folder. It is designed for users who want to scale AI image creation and organize results automatically in Google Drive.

The workflow logic is grouped into the following functional blocks:

- **1.1 Input Reception:** Manually triggers the workflow and fetches image prompts from a Google Sheets document.
- **1.2 Prompt Duplication:** Duplicates each prompt to generate multiple image variants per prompt.
- **1.3 Image Generation:** Loops through each duplicated prompt to call Freepik’s AI image generation API.
- **1.4 Post-Processing:** Splits multiple images returned per API call, converts base64 image data into binary files, and uploads each image to Google Drive.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
Starts the workflow manually and retrieves prompts from a Google Sheets document.

**Nodes Involved:**  
- Start Workflow (Manual Trigger)  
- Get Prompt from Google Sheet (Google Sheets)

**Node Details:**

- **Start Workflow**  
  - Type: Manual Trigger  
  - Role: Entry point to manually start the workflow.  
  - Configuration: No parameters; triggers on user action.  
  - Inputs: None  
  - Outputs: Connects to "Get Prompt from Google Sheet"  
  - Edge Cases: None, except user not triggering the start.  

- **Get Prompt from Google Sheet**  
  - Type: Google Sheets  
  - Role: Reads prompts (and associated metadata like Name) from a Google Sheets spreadsheet.  
  - Configuration:  
    - Document ID: ID of the Google Sheets file containing prompts.  
    - Sheet Name: "Sheet1" (or specified sheet)  
    - Operation: Read rows  
    - Credentials: Google Sheets OAuth2 credentials required  
  - Key Expressions: None beyond standard field mapping.  
  - Inputs: From "Start Workflow"  
  - Outputs: To "Double Output"  
  - Edge Cases:  
    - Authentication failure if credentials expire.  
    - Empty or malformed sheet data.  
    - API rate limits from Google Sheets.  

---

#### 2.2 Prompt Duplication

**Overview:**  
Duplicates each prompt to create multiple runs (e.g., two variations per prompt), enabling generation of multiple images per input.

**Nodes Involved:**  
- Double Output (Code)  
- Loop (SplitInBatches)

**Node Details:**

- **Double Output**  
  - Type: Code (JavaScript)  
  - Role: Creates two copies of each prompt item, each tagged with a "run" number (1 or 2).  
  - Configuration:  
    - JavaScript code duplicates the single input item into two outputs with run numbers.  
  - Key Code Snippet:  
    ```javascript
    const original = items[0].json;
    return [
      { json: { ...original, run: 1 } },
      { json: { ...original, run: 2 } },
    ];
    ```  
  - Inputs: From "Get Prompt from Google Sheet" (one item at a time)  
  - Outputs: To "Loop" node  
  - Edge Cases:  
    - If input item is missing fields, duplication will propagate errors downstream.  

- **Loop**  
  - Type: SplitInBatches  
  - Role: Processes prompts one by one or in batches to avoid overloading the API and control flow.  
  - Configuration: Default batch size (not specified, so defaults apply).  
  - Inputs: From "Double Output"  
  - Outputs: To "Create Image"  
  - Edge Cases:  
    - Large batch size could cause timeout or API rate limits.  
    - Network interruptions could halt batch processing.  

---

#### 2.3 Image Generation

**Overview:**  
Sends each prompt to Freepik’s AI text-to-image API to generate images.

**Nodes Involved:**  
- Create Image (HTTP Request)  
- Split Responses (Split Out)

**Node Details:**

- **Create Image**  
  - Type: HTTP Request  
  - Role: Calls Freepik API to generate images based on the prompt.  
  - Configuration:  
    - Method: POST  
    - URL: `https://api.freepik.com/v1/ai/text-to-image`  
    - Authentication: Generic HTTP Header Auth with API key/token.  
    - Body Parameter: `prompt` set dynamically to `={{ $json.Prompt }}` from input item.  
  - Inputs: From "Loop"  
  - Outputs: To "Split Responses"  
  - Edge Cases:  
    - API authentication errors.  
    - API rate limiting.  
    - Malformed prompts causing API errors.  
    - Network timeouts.  

- **Split Responses**  
  - Type: Split Out  
  - Role: Separates multiple images returned in the `data` array from the API response into individual items for further processing.  
  - Configuration:  
    - Field to split out: `data` (array of images)  
  - Inputs: From "Create Image"  
  - Outputs: To "Convert to File"  
  - Edge Cases:  
    - Empty or missing `data` field could cause no outputs.  
    - Unexpected response structure.  

---

#### 2.4 Post-Processing and Upload

**Overview:**  
Converts the base64 image data from API into binary files suitable for upload, then uploads each image to a specified Google Drive folder.

**Nodes Involved:**  
- Convert to File (Convert to File)  
- Upload Image to Google Drive (Google Drive)

**Node Details:**

- **Convert to File**  
  - Type: Convert to File  
  - Role: Converts base64 encoded image strings into binary file data for upload.  
  - Configuration:  
    - Operation: toBinary  
    - Source Property: `base64` (the property holding base64 encoded image data)  
  - Inputs: From "Split Responses"  
  - Outputs: To "Upload Image to Google Drive"  
  - Edge Cases:  
    - Missing or invalid base64 data will cause conversion failure.  

- **Upload Image to Google Drive**  
  - Type: Google Drive  
  - Role: Uploads the binary image files to a specified Google Drive folder with a dynamic name.  
  - Configuration:  
    - Operation: Upload  
    - File Name: Dynamically constructed as `Image - {{ Name }} - {{ run }}`, combining the prompt's name and run number.  
    - Drive: "My Drive"  
    - Folder ID: Predefined Google Drive folder ID (configured in parameters)  
    - Credentials: Google Drive OAuth2 credentials required  
  - Inputs: From "Convert to File"  
  - Outputs: To "Loop" node (to continue loop processing)  
  - Edge Cases:  
    - Authentication failures.  
    - Insufficient permissions on Google Drive folder.  
    - Upload size limits or network issues.  

---

### 3. Summary Table

| Node Name                 | Node Type          | Functional Role                         | Input Node(s)                 | Output Node(s)              | Sticky Note                                                                                                                             |
|---------------------------|--------------------|---------------------------------------|------------------------------|-----------------------------|-----------------------------------------------------------------------------------------------------------------------------------------|
| Start Workflow            | Manual Trigger     | Entry point, manual start              | None                         | Get Prompt from Google Sheet | No sticky note                                                                                                                          |
| Get Prompt from Google Sheet | Google Sheets     | Reads prompts from Google Sheets       | Start Workflow               | Double Output               | Configuration details for Google Sheets node, including document ID and sheet name                                                      |
| Double Output             | Code               | Duplicates each prompt into two runs   | Get Prompt from Google Sheet | Loop                        | Code snippet provided for duplication; purpose is to create multiple prompt variations                                                  |
| Loop                      | SplitInBatches     | Controls batch processing of prompts   | Double Output                | Create Image                | No sticky note                                                                                                                          |
| Create Image              | HTTP Request       | Calls Freepik AI image generation API | Loop                        | Split Responses             | Detailed HTTP request config with API URL, POST method, and authentication                                                             |
| Split Responses           | Split Out          | Splits multiple images from API response | Create Image               | Convert to File             | Purpose to separate images within the API response array                                                                                |
| Convert to File           | Convert to File    | Converts base64 image to binary file   | Split Responses              | Upload Image to Google Drive | Configured to convert `base64` property to binary                                                                                       |
| Upload Image to Google Drive | Google Drive     | Uploads images to Google Drive folder  | Convert to File              | Loop                        | Dynamic naming of uploaded file; configured folder ID and Drive; requires Google Drive OAuth2 credentials                               |
| Sticky Note16             | Sticky Note        | Support contact info                    | None                         | None                        | Contact email and LinkedIn for help                                                                                                    |
| Sticky Note               | Sticky Note        | Node configuration details summary     | None                         | None                        | Configuration details for key nodes including Start, Sheets, Code, and link to Google Sheet template                                   |
| Sticky Note1              | Sticky Note        | Detailed node configs for HTTP request and uploads | None                    | None                        | Detailed config info for Create Image, Split Responses, Convert to File, Upload Image nodes                                            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a Manual Trigger node:**  
   - Name: `Start Workflow`  
   - No configuration needed.

3. **Add a Google Sheets node:**  
   - Name: `Get Prompt from Google Sheet`  
   - Operation: Read rows  
   - Document ID: Set to your Google Sheets document ID containing prompts  
   - Sheet Name: Set to your sheet name (e.g., "Sheet1")  
   - Credentials: Select or create Google Sheets OAuth2 credentials.

4. **Connect `Start Workflow` to `Get Prompt from Google Sheet`.**

5. **Add a Code node:**  
   - Name: `Double Output`  
   - Language: JavaScript  
   - Code:  
     ```javascript
     const original = items[0].json;
     return [
       { json: { ...original, run: 1 } },
       { json: { ...original, run: 2 } },
     ];
     ```
   - Purpose: Duplicate each prompt to generate two variations.

6. **Connect `Get Prompt from Google Sheet` to `Double Output`.**

7. **Add a SplitInBatches node:**  
   - Name: `Loop`  
   - Default batch size (or set as needed to control API calls).  

8. **Connect `Double Output` to `Loop`.**

9. **Add an HTTP Request node:**  
   - Name: `Create Image`  
   - Method: POST  
   - URL: `https://api.freepik.com/v1/ai/text-to-image`  
   - Authentication: Generic HTTP Header Auth  
   - Credentials: Create or select credentials with Freepik API key in HTTP header  
   - Body Parameters: Add parameter named `prompt` with expression: `={{ $json.Prompt }}`  
   - Send Body: true  

10. **Connect `Loop` to `Create Image`.**

11. **Add a Split Out node:**  
    - Name: `Split Responses`  
    - Field to split out: `data` (the array of images from API response)

12. **Connect `Create Image` to `Split Responses`.**

13. **Add a Convert to File node:**  
    - Name: `Convert to File`  
    - Operation: `toBinary`  
    - Source Property: `base64` (which contains the base64 encoded image string)

14. **Connect `Split Responses` to `Convert to File`.**

15. **Add a Google Drive node:**  
    - Name: `Upload Image to Google Drive`  
    - Operation: Upload  
    - Drive ID: Select "My Drive" or your desired drive  
    - Folder ID: Set to your target Google Drive folder ID for uploads  
    - File Name: Set to expression:  
      ```
      Image - {{ $('Get Prompt from Google Sheet').item.json.Name }} - {{ $('Double Output').item.json.run }}
      ```  
    - Credentials: Select or create Google Drive OAuth2 credentials.

16. **Connect `Convert to File` to `Upload Image to Google Drive`.**

17. **Connect `Upload Image to Google Drive` back to `Loop`** to continue batch processing.

18. **Optionally add sticky notes with configuration details and contact info for maintainers.**

---

### 5. General Notes & Resources

| Note Content                                                                                         | Context or Link                                                                                                             |
|----------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------|
| Contact for help or customization: robert@ynteractive.com, LinkedIn: https://www.linkedin.com/in/robert-breen-29429625/ | Support contact information provided in sticky note16 for user assistance.                                                  |
| Google Sheets template for prompts: https://docs.google.com/spreadsheets/d/1_u9IxEZINcwKQB15Rfx7C1hM71zeDST58Fz3nRHTCUY/edit?usp=sharing | Prebuilt Google Sheets document with sample prompts to use with this workflow, referenced in sticky note.                    |
| Freepik API documentation required for authentication setup and parameter details.                  | Refer to Freepik AI API docs for authentication schemes and parameter customization.                                        |

---

**Disclaimer:**  
The provided text is exclusively derived from an automated workflow created with n8n, a tool for integration and automation. This workflow respects all applicable content policies and contains no illegal, offensive, or protected elements. All handled data is legal and publicly accessible.