Create Consistent AI Characters with Google Nano Banana & Upscaling via Kie.ai

https://n8nworkflows.xyz/workflows/create-consistent-ai-characters-with-google-nano-banana---upscaling-via-kie-ai-8492


# Create Consistent AI Characters with Google Nano Banana & Upscaling via Kie.ai

---

### 1. Workflow Overview

This workflow automates the creation of consistent AI-generated character images using the Google Nano Banana model, followed by image upscaling through Kie.ai. It targets creators and developers who want to generate cinematic, photorealistic character portraits based on textual prompts, manage assets on Google Drive, and track progress via Google Sheets. The workflow is divided into three main logical blocks:

- **1.1 Initialization and Folder Setup:** Prepares the working environment by generating timestamped folder names, creating folders in Google Drive, and setting up a Google Sheets spreadsheet to track image generation status.

- **1.2 AI Story and Image Generation:** Uses LangChain with OpenAI GPT-4 to generate detailed prompts for AI image creation, submits tasks to Kie.ai’s Nano Banana model, polls for completion, extracts generated image URLs, and updates the Google Sheet with prompt and status.

- **1.3 Image Upscaling and Management:** Takes generated images, submits them to Kie.ai’s upscaling model, polls for completion, downloads the upscaled images, uploads them to Google Drive, moves the files into the correct folder, and updates the Google Sheet with the final image URLs and status.

---

### 2. Block-by-Block Analysis

#### 2.1 Initialization and Folder Setup

**Overview:**  
This block initializes the workflow by creating a dated folder in Google Drive and a corresponding Google Sheets spreadsheet to log prompts, image URLs, and statuses. It prepares the environment for storing generated assets and tracking progress.

**Nodes Involved:**  
- When clicking ‘Execute workflow’ (Manual Trigger)  
- Folder Name (Code)  
- Create folder (Google Drive)  
- Create spreadsheet (Google Sheets)  
- Move Sheet (Google Drive)  
- Edit Fields (Set)  
- Add Fields (Google Sheets)

**Node Details:**

- **When clicking ‘Execute workflow’**  
  - Type: Manual Trigger  
  - Role: Start point to manually trigger the workflow.  
  - Inputs: None  
  - Outputs: Triggers the next node.

- **Folder Name**  
  - Type: Code  
  - Role: Generates a human-readable folder name with the current date and time in format “Post HH:MM AM/PM DD Month YYYY”.  
  - Key logic: Uses JavaScript date functions to format the timestamp.  
  - Inputs: Trigger from Manual Trigger node  
  - Outputs: JSON object with `folderName`.  
  - Edge cases: None significant; system time-dependent.

- **Create folder**  
  - Type: Google Drive Folder Create  
  - Role: Creates a new folder in Google Drive under a fixed parent folder ID.  
  - Config: Uses OAuth2 credentials for Google Drive. Folder name is dynamic from `Folder Name` node output.  
  - Input: `folderName` from previous node  
  - Output: JSON with folder ID and metadata for downstream use.  
  - Edge cases: API rate limits, permission errors.

- **Create spreadsheet**  
  - Type: Google Sheets Create  
  - Role: Creates a new Google Sheets spreadsheet with the same name as the folder name.  
  - Config: Uses OAuth2 credentials for Google Sheets.  
  - Input: Folder name passed as spreadsheet title.  
  - Output: Spreadsheet metadata including spreadsheet ID and first sheet ID.

- **Move Sheet**  
  - Type: Google Drive File Move  
  - Role: Moves the newly created Google Sheets file into the newly created folder.  
  - Config: Uses spreadsheet ID from previous node and folder ID from `Create folder`.  
  - Edge cases: Permissions, invalid IDs.

- **Edit Fields**  
  - Type: Set  
  - Role: Initializes empty fields (`Prompt`, `Temp Image Url`, `Status`) to prepare for appending data to the sheet.  
  - Output: JSON with empty placeholders.

- **Add Fields**  
  - Type: Google Sheets Append  
  - Role: Appends the initialized fields as a new row in the spreadsheet.  
  - Config: Uses spreadsheet and sheet IDs from previous nodes. Uses OAuth2 credentials for Google Sheets.  
  - Edge cases: Sheet access errors, quota limits.

---

#### 2.2 AI Story and Image Generation

**Overview:**  
This block generates detailed prompts for AI image creation using OpenAI GPT-4 via LangChain, submits image generation tasks to Kie.ai’s Nano Banana model, polls for task completion, extracts image URLs, and updates the sheet with status.

**Nodes Involved:**  
- Story Creator Agent (LangChain LLM Chain)  
- OpenAI Chat Model3 (LangChain OpenAI LLM)  
- Json parser4 (LangChain output parser)  
- Code1 (Code)  
- Story Status Update (Google Sheets Update)  
- Create Task (no callback) (HTTP Request)  
- Set TaskId (Set)  
- Wait 1 Min (Wait)  
- Poll Task (HTTP Request)  
- Switch (Switch)  
- Get ResultUrls (Code)

**Node Details:**

- **Story Creator Agent**  
  - Type: LangChain LLM Chain  
  - Role: Generates a cinematic, photorealistic portrait prompt based on input template and context.  
  - Config: Uses a static prompt template with placeholders describing scene, styling, and mood.  
  - Inputs: Receives structured input from previous nodes.  
  - Outputs: Parsed prompt text.  
  - Edge cases: LLM API errors, malformed input.

- **OpenAI Chat Model3**  
  - Type: LangChain OpenAI Chat Model  
  - Role: Provides GPT-4.1-mini model for natural language generation.  
  - Config: Uses OpenAI credentials.  
  - Input: Input prompt from `Story Creator Agent`.  
  - Output: Raw AI response.  
  - Edge cases: API quota, timeout, rate limits.

- **Json parser4**  
  - Type: LangChain output parser  
  - Role: Parses structured JSON output from LLM for prompt extraction.  
  - Input: Raw LLM output.  
  - Output: JSON with `prompt` property.  
  - Edge cases: Parsing errors if output is malformed.

- **Code1**  
  - Type: Code  
  - Role: Cleans and normalizes prompt strings, unescaping JSON-style escape characters and removing extra whitespace.  
  - Input: Output of parser.  
  - Output: Cleaned prompt string.  
  - Edge cases: Unexpected JSON formatting.

- **Story Status Update**  
  - Type: Google Sheets Update  
  - Role: Updates the status and prompt fields in the sheet to indicate prompt creation.  
  - Input: Cleaned prompt from `Code1`.  
  - Config: Specifies row number 2 for updating the sheet.  
  - Edge cases: Sheet write permission errors.

- **Create Task (no callback)**  
  - Type: HTTP Request  
  - Role: Sends a POST request to Kie.ai API to create an image generation task with the Nano Banana model.  
  - Config: Uses Bearer token authorization (replace `YOUR_API_KEY`).  
  - Input: JSON body includes the cleaned prompt and a fixed array of 5 image URLs as references. Output format: PNG, image size auto.  
  - Output: Task creation response with taskId.  
  - Edge cases: Authentication failure, network errors.

- **Set TaskId**  
  - Type: Set  
  - Role: Extracts and stores `taskId` from `Create Task` response for polling.  
  - Output: JSON with `taskId`.

- **Wait 1 Min**  
  - Type: Wait  
  - Role: Delays workflow execution for 1 minute before polling for task completion.  
  - Edge cases: None.

- **Poll Task**  
  - Type: HTTP Request  
  - Role: Polls Kie.ai API job status by `taskId`.  
  - Input: Query parameter `taskId` from `Set TaskId`.  
  - Output: Status response with job state and result JSON.  
  - Edge cases: Timeout, API errors.

- **Switch**  
  - Type: Switch  
  - Role: Routes flow based on job status:  
    - `fail state` if state is "fail"  
    - `false` if resultJson is empty (continue waiting)  
    - `Success` if resultJson includes success message.  
  - Output: Branches for error handling, waiting, or next step.

- **Get ResultUrls**  
  - Type: Code  
  - Role: Extracts the array of generated image URLs from the result JSON string.  
  - Input: Polling result JSON.  
  - Output: JSON containing `resultUrls` array.  
  - Edge cases: JSON parsing errors.

---

#### 2.3 Image Upscaling and Management

**Overview:**  
This block upscales the generated images using Kie.ai’s Nano Banana upscale model, polls for completion, downloads the upscaled images, uploads them to Google Drive, moves them to the correct folder, and updates the Google Sheet with the final image URLs and status.

**Nodes Involved:**  
- Upscale Image (HTTP Request)  
- Set TaskId for upscale (Set)  
- Wait 1 Min1 (Wait)  
- Get Upscaled Image (HTTP Request)  
- Switch1 (Switch)  
- Get ResultUrls from UpScale (Code)  
- Get Binary (HTTP Request)  
- Upload file (Google Drive)  
- Move Images (Google Drive)  
- Update Image Status (Google Sheets Update)

**Node Details:**

- **Upscale Image**  
  - Type: HTTP Request  
  - Role: Submits an upscaling request to Kie.ai with scale 4x and face enhancement enabled.  
  - Input: Uses the first generated image URL from `Get ResultUrls`.  
  - Output: Task creation response with upscale `taskId`.  
  - Edge cases: API auth errors.

- **Set TaskId for upscale**  
  - Type: Set  
  - Role: Stores the upscale taskId for polling.  
  - Output: JSON with `taskId`.

- **Wait 1 Min1**  
  - Type: Wait  
  - Role: Waits 1 minute before polling upscale task.  
  - Edge cases: None.

- **Get Upscaled Image**  
  - Type: HTTP Request  
  - Role: Polls Kie.ai API for upscale task status.  
  - Input: Uses upscale `taskId`.  
  - Output: Status data including result JSON.  
  - Edge cases: API errors, timeouts.

- **Switch1**  
  - Type: Switch  
  - Role: Routes based on upscale task status, similar to the earlier switch: fail, waiting, or success.  
  - Outputs: Branch to re-submit upscale, wait, or proceed.

- **Get ResultUrls from UpScale**  
  - Type: Code  
  - Role: Parses upscale result JSON to extract upscaled image URLs.  
  - Output: JSON containing `resultUrls`.

- **Get Binary**  
  - Type: HTTP Request  
  - Role: Downloads the upscaled image file from extracted URLs.  
  - Output: Binary file data.

- **Upload file**  
  - Type: Google Drive Upload  
  - Role: Uploads the downloaded binary file to Google Drive root or specified folder.  
  - Input: File binary and metadata.  
  - Output: File metadata including file ID.

- **Move Images**  
  - Type: Google Drive Move  
  - Role: Moves uploaded image file into the working folder created at workflow start.  
  - Input: File ID from upload, folder ID from initial folder creation.  
  - Edge cases: Permissions.

- **Update Image Status**  
  - Type: Google Sheets Update  
  - Role: Updates the spreadsheet row with status "image created" and the URL of the uploaded image.  
  - Input: Row number, status, and image URL.  
  - Output: Confirmation of sheet update.

---

### 3. Summary Table

| Node Name                | Node Type                         | Functional Role                          | Input Node(s)               | Output Node(s)             | Sticky Note                                                                                                                                                                                                                                                                                                                                                                   |
|--------------------------|----------------------------------|---------------------------------------|-----------------------------|----------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger                   | Workflow entry point                   | None                        | Folder Name                |                                                                                                                                                                                                                                                                                                                                                                               |
| Folder Name              | Code                             | Generates dated folder name            | When clicking ‘Execute workflow’ | Create folder             |                                                                                                                                                                                                                                                                                                                                                                               |
| Create folder            | Google Drive Folder              | Creates folder in Google Drive         | Folder Name                  | Create spreadsheet         | ## Create Folder and File                                                                                                                                                                                                                                                                                                                                                     |
| Create spreadsheet       | Google Sheets Create             | Creates spreadsheet for logging        | Create folder                | Move Sheet                 | ## Create Folder and File                                                                                                                                                                                                                                                                                                                                                     |
| Move Sheet               | Google Drive File Move           | Moves spreadsheet into folder           | Create spreadsheet           | Edit Fields                | ## Create Folder and File                                                                                                                                                                                                                                                                                                                                                     |
| Edit Fields              | Set                             | Initializes empty row fields             | Move Sheet                   | Add Fields                 | ## Create Folder and File                                                                                                                                                                                                                                                                                                                                                     |
| Add Fields               | Google Sheets Append             | Appends initialized fields as new row   | Edit Fields                  | Story Creator Agent        | ## Create Story<br>## Update Status in Sheet                                                                                                                                                                                                                                                                                                                                 |
| Story Creator Agent      | LangChain LLM Chain              | Generates detailed AI image prompt      | Add Fields                   | Code1                      | ## Create Story<br>## Update Status in Sheet                                                                                                                                                                                                                                                                                                                                 |
| OpenAI Chat Model3       | LangChain OpenAI Chat Model      | Provides GPT-4.1-mini for prompt gen    | Story Creator Agent          | Story Creator Agent        | ## Create Story<br>## Update Status in Sheet                                                                                                                                                                                                                                                                                                                                 |
| Json parser4             | LangChain Output Parser          | Parses JSON output from LLM              | OpenAI Chat Model3           | Story Creator Agent        | ## Create Story<br>## Update Status in Sheet                                                                                                                                                                                                                                                                                                                                 |
| Code1                    | Code                            | Cleans and normalizes prompt strings     | Story Creator Agent          | Story Status Update        | ## Create Story<br>## Update Status in Sheet                                                                                                                                                                                                                                                                                                                                 |
| Story Status Update      | Google Sheets Update             | Updates sheet status and prompt          | Code1                       | Create Task (no callback)  | ## Create Story<br>## Update Status in Sheet                                                                                                                                                                                                                                                                                                                                 |
| Create Task (no callback)| HTTP Request                    | Submits Nano Banana image generation task| Story Status Update          | Set TaskId                 | ## Create Image<br>## with NanoBanana                                                                                                                                                                                                                                                                                                                                         |
| Set TaskId               | Set                             | Stores taskId for polling                 | Create Task (no callback)    | Wait 1 Min                 | ## Create Image<br>## with NanoBanana                                                                                                                                                                                                                                                                                                                                         |
| Wait 1 Min               | Wait                            | Waits 1 minute before polling             | Set TaskId                   | Poll Task                  | ## Create Image<br>## with NanoBanana                                                                                                                                                                                                                                                                                                                                         |
| Poll Task                | HTTP Request                    | Polls Kie.ai for task status              | Wait 1 Min                  | Switch                     | ## Create Image<br>## with NanoBanana                                                                                                                                                                                                                                                                                                                                         |
| Switch                   | Switch                          | Routes workflow based on task status     | Poll Task                   | Create Task (no callback), Wait 1 Min, Get ResultUrls | ## Create Image<br>## with NanoBanana                                                                                                                                                                                                                                                                                                                                         |
| Get ResultUrls           | Code                            | Extracts generated image URLs             | Switch (Success branch)      | Upscale Image              | ## Create Image<br>## with NanoBanana                                                                                                                                                                                                                                                                                                                                         |
| Upscale Image            | HTTP Request                    | Submits image upscale task                | Get ResultUrls               | Set TaskId for upscale     | ## Upscale Image                                                                                                                                                                                                                                                                                                                                                               |
| Set TaskId for upscale   | Set                             | Stores upscale taskId                      | Upscale Image               | Wait 1 Min1                | ## Upscale Image                                                                                                                                                                                                                                                                                                                                                               |
| Wait 1 Min1              | Wait                            | Waits 1 minute before polling upscale     | Set TaskId for upscale       | Get Upscaled Image         | ## Upscale Image                                                                                                                                                                                                                                                                                                                                                               |
| Get Upscaled Image       | HTTP Request                    | Polls Kie.ai for upscale task status      | Wait 1 Min1                 | Switch1                    | ## Upscale Image                                                                                                                                                                                                                                                                                                                                                               |
| Switch1                  | Switch                          | Routes based on upscale task status       | Get Upscaled Image          | Upscale Image, Set TaskId for upscale, Get ResultUrls from UpScale | ## Upscale Image                                                                                                                                                                                                                                                                                                                                                               |
| Get ResultUrls from UpScale | Code                          | Extracts URLs from upscale task result    | Switch1                     | Get Binary                 | ## Upscale Image                                                                                                                                                                                                                                                                                                                                                               |
| Get Binary               | HTTP Request                    | Downloads upscaled image                   | Get ResultUrls from UpScale | Upload file                | ## Upscale Image                                                                                                                                                                                                                                                                                                                                                               |
| Upload file              | Google Drive Upload             | Uploads image file to Google Drive         | Get Binary                  | Move Images                | ## Upscale Image                                                                                                                                                                                                                                                                                                                                                               |
| Move Images              | Google Drive Move               | Moves uploaded image into working folder   | Upload file                 | Update Image Status        | ## Upscale Image                                                                                                                                                                                                                                                                                                                                                               |
| Update Image Status      | Google Sheets Update             | Updates sheet with final image URL and status | Move Images               | None                      | ## Upscale Image                                                                                                                                                                                                                                                                                                                                                               |
| Sticky Note              | Sticky Note                    | Comment block                             | None                        | None                      | ## Create Folder and File                                                                                                                                                                                                                                                                                                                                                     |
| Sticky Note3             | Sticky Note                    | Comment block                             | None                        | None                      | ## Create Story<br>## Update Status in Sheet                                                                                                                                                                                                                                                                                                                                 |
| Sticky Note4             | Sticky Note                    | Comment block                             | None                        | None                      | ## Create Image<br>## with NanoBanana                                                                                                                                                                                                                                                                                                                                         |
| Sticky Note5             | Sticky Note                    | Comment block                             | None                        | None                      | ## Upscale Image                                                                                                                                                                                                                                                                                                                                                               |
| Sticky Note7             | Sticky Note                    | Author contact and expertise info          | None                        | None                      | Muhammad Farooq Iqbal - Automation Expert & n8n Creator. Contact: mfarooqiqbal143@gmail.com, +923036991118, [LinkedIn](https://linkedin.com/in/muhammadfarooqiqbal), [Portfolio](https://mfarooqone.github.io/n8n/)                                                                                                                                                            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger node** named `When clicking ‘Execute workflow’`.

2. **Add a Code node** named `Folder Name`.  
   - Paste JavaScript to generate a timestamped folder name in format:  
     `Post HH:MM AM/PM DD Month YYYY`.  
   - Output: `{ folderName: string }`

3. **Add a Google Drive node** named `Create folder`.  
   - Operation: Create Folder  
   - Folder Name: Use expression from `Folder Name` node output.  
   - Parent Folder ID: Fixed ID (replace with your own).  
   - Credentials: Google Drive OAuth2.

4. **Add a Google Sheets node** named `Create spreadsheet`.  
   - Operation: Create Spreadsheet  
   - Title: Use the folder name from `Folder Name` node.  
   - Credentials: Google Sheets OAuth2.

5. **Add a Google Drive node** named `Move Sheet`.  
   - Operation: Move File  
   - File ID: Spreadsheet ID from `Create spreadsheet`.  
   - Destination Folder ID: Folder ID from `Create folder`.  
   - Credentials: Google Drive OAuth2.

6. **Add a Set node** named `Edit Fields`.  
   - Initialize empty fields: `Prompt`, `Temp Image Url`, `Status` all as empty strings.

7. **Add a Google Sheets Append node** named `Add Fields`.  
   - Append row to the created spreadsheet sheet.  
   - Map fields automatically.  
   - Credentials: Google Sheets OAuth2.

8. **Add a LangChain LLM Chain node** named `Story Creator Agent`.  
   - Configure prompt template describing photorealistic cinematic portrait with placeholders.  
   - Enable output parser (JSON).

9. **Add LangChain OpenAI Chat Model node** named `OpenAI Chat Model3`.  
   - Model: `gpt-4.1-mini` or equivalent.  
   - Credentials: OpenAI API.

10. **Add a LangChain output parser node** named `Json parser4`.  
    - Provide JSON schema example: `{ "prompt": "detailed scene 1 description" }`.

11. **Connect `OpenAI Chat Model3` to `Story Creator Agent` as AI language model input. Connect `Story Creator Agent` to `Json parser4` as output parser.**

12. **Add a Code node** named `Code1`.  
    - Paste JavaScript to clean and normalize prompt strings, unescaping JSON-style escapes and collapsing whitespace.

13. **Add a Google Sheets Update node** named `Story Status Update`.  
    - Update row 2 with `Prompt` and `Status` = "prompt created".  
    - Credentials: Google Sheets OAuth2.

14. **Add an HTTP Request node** named `Create Task (no callback)`.  
    - POST to `https://api.kie.ai/api/v1/jobs/createTask`.  
    - JSON Body includes:  
      - model: `"google/nano-banana-edit"`  
      - input: prompt from `Code1`, fixed array of 5 reference image URLs, output_format `png`, image_size `auto`.  
    - Headers: Authorization Bearer token (replace `YOUR_API_KEY`).  
    - Credentials: None (use header auth).

15. **Add a Set node** named `Set TaskId`.  
    - Extract `taskId` from `Create Task` response.

16. **Add a Wait node** named `Wait 1 Min`.  
    - Duration: 1 minute.

17. **Add an HTTP Request node** named `Poll Task`.  
    - GET `https://api.kie.ai/api/v1/jobs/recordInfo` with query parameter `taskId`.  
    - Authorization header as above.

18. **Add a Switch node** named `Switch`.  
    - Route based on `data.state` and `data.resultJson`:  
      - Output "fail state" if `state == "fail"`  
      - Output "false" if `resultJson` is empty (continue waiting)  
      - Output "Success" if `resultJson` contains success string.

19. **Add a Code node** named `Get ResultUrls`.  
    - Extract and parse `resultJson` to get array of `resultUrls`.

20. **Add an HTTP Request node** named `Upscale Image`.  
    - POST to Kie.ai upscale endpoint with:  
      - model: `"nano-banana-upscale"`  
      - input: first image URL from `Get ResultUrls`, scale 4, face_enhance true.  
    - Authorization header.

21. **Add a Set node** named `Set TaskId for upscale`.  
    - Extract upscale `taskId`.

22. **Add a Wait node** named `Wait 1 Min1`.  
    - Duration: 1 minute.

23. **Add an HTTP Request node** named `Get Upscaled Image`.  
    - Poll upscale task status as in step 17.

24. **Add a Switch node** named `Switch1`.  
    - Same logic as `Switch` node.

25. **Add a Code node** named `Get ResultUrls from UpScale`.  
    - Extract `resultUrls` from upscale task result JSON.

26. **Add an HTTP Request node** named `Get Binary`.  
    - Download the first upscaled image URL, response binary.

27. **Add a Google Drive node** named `Upload file`.  
    - Upload binary file to Google Drive root or specified folder.  
    - Credentials: Google Drive OAuth2.

28. **Add a Google Drive node** named `Move Images`.  
    - Move uploaded file to folder created in step 3.

29. **Add a Google Sheets Update node** named `Update Image Status`.  
    - Update row 2 with status "image created" and `Temp Image Url` with uploaded file URL.

30. **Add Sticky Note nodes** as needed for documentation and annotation in the editor.

31. **Connect all nodes following the original workflow’s connections, ensuring error handling via Switch nodes and wait loops for polling.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                     | Context or Link                                                                                   |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Workflow created by Muhammad Farooq Iqbal, experienced automation expert and n8n creator with 500+ downloads of workflow templates. Contact via email, phone, LinkedIn, and portfolio link.                                                                                                                                                                      | Contact info in Sticky Note7: mfarooqiqbal143@gmail.com, +923036991118, [LinkedIn](https://linkedin.com/in/muhammadfarooqiqbal), [Portfolio](https://mfarooqone.github.io/n8n/) |
| Ensure to replace all placeholder `YOUR_API_KEY` with valid Kie.ai API keys for authentication.                                                                                                                                                                                                                                                                | API authentication requirement.                                                                 |
| The workflow assumes fixed Google Drive folder ID `1GND1exvlAXTzESNvWmFs6FFMx90KLzlC` as a parent folder; replace with your own folder ID to organize files properly.                                                                                                                                                                                             | Google Drive folder organization.                                                               |
| This workflow uses LangChain nodes for AI prompt generation and parsing. Requires n8n LangChain nodes installed and API credentials configured.                                                                                                                                                                                                               | LangChain node requirement.                                                                      |
| Polling intervals are set to 1 minute; adjust wait times if API response times vary to improve performance and avoid timeouts.                                                                                                                                                                                                                                  | Performance tuning suggestion.                                                                   |
| The workflow uses specific image URLs as references for consistent character generation; update these URLs to match your desired reference images.                                                                                                                                                                                                             | Image reference customization.                                                                   |

---

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

---