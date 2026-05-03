Generate AI Stock Images with Flux1-schnell, Metadata Tagging & Google Integration

https://n8nworkflows.xyz/workflows/generate-ai-stock-images-with-flux1-schnell--metadata-tagging---google-integration-4445


# Generate AI Stock Images with Flux1-schnell, Metadata Tagging & Google Integration

### 1. Workflow Overview

This workflow automates the generation, processing, and management of AI-generated stock images using the Flux1-schnell model. It takes prompts from a Google Sheet, generates images via an AI image generation service, analyzes and tags images with metadata, uploads them to Google Drive, logs results back into Google Sheets, and notifies via Telegram. The workflow is designed for use cases involving bulk AI image production with automated metadata tagging and cloud storage integration, aimed at streamlining stock image creation and cataloging.

The workflow’s logic is divided into the following main blocks:

- **1.1 Scheduled Data Retrieval & Prompt Preparation:** Periodic triggers read prompts and date filters from Google Sheets, preparing input data for image generation.
- **1.2 Prompt Processing & AI Image Generation:** Prompts are refined, split, and passed to the Flux1-schnell AI model to generate images asynchronously with controlled timing.
- **1.3 Image Downloading & Processing:** Generated images are downloaded, optionally resized and composed, then analyzed using OpenAI for metadata extraction.
- **1.4 Metadata Parsing & Catalog Updating:** Extracted metadata is parsed, formatted, and recorded into Google Sheets for tracking.
- **1.5 Google Drive Folder Management & Upload:** The workflow verifies or creates necessary Drive folders and sheets, then uploads generated images accordingly.
- **1.6 Notifications & Error Handling:** Progress and errors are logged and communicated via Telegram, with error triggers capturing failures.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Data Retrieval & Prompt Preparation

**Overview:**  
Periodically triggers the workflow to retrieve date-filtered prompts from Google Sheets, preparing them for AI image generation.

**Nodes Involved:**  
- Schedule Trigger  
- Set Date Info  
- filter data date  
- Google Sheets  
- Select Prompt  
- Check the folder  
- Folder Exists?  
- Check sheet  
- Sheet Exists?  
- Set Folder ID sheet  
- Set Folder ID drive folder  
- Create Folder for images  
- Create New Sheet  
- Check the folder  
- Check sheet  

**Node Details:**

- **Schedule Trigger**  
  - Type: Trigger node that initiates execution on schedule.  
  - Config: Default schedule (unspecified in JSON).  
  - Role: Start point for scheduled runs.

- **Set Date Info**  
  - Type: Code node.  
  - Role: Sets or calculates date parameters used to filter prompts.  
  - Key variables: Date ranges for filtering.  
  - Input: Trigger output.  
  - Output: Date info passed to next node.

- **filter data date**  
  - Type: Code node.  
  - Role: Filters Google Sheets data according to date info.  
  - Input: Date info and sheet data.  
  - Potential errors: Incorrect date format or empty data.

- **Google Sheets**  
  - Type: Google Sheets node (v4.5).  
  - Role: Reads prompt data from a configured spreadsheet.  
  - Config: Spreadsheet and sheet names set in parameters (not shown).  
  - Potential errors: Authentication, sheet not found, API limits.

- **Select Prompt**  
  - Type: Code node.  
  - Role: Processes/filter prompts for image generation.  
  - Connections: Outputs to Check the folder and Check sheet nodes.

- **Check the folder**  
  - Type: Google Drive node.  
  - Role: Checks if image folder exists in Drive.  
  - OnError: Continue.  
  - Output: Passes to Folder Exists? node.

- **Folder Exists?**  
  - Type: If node.  
  - Role: Conditional branch based on folder presence.  
  - True: Pass to Set Folder ID drive folder.  
  - False: Pass to Create Folder for images.

- **Check sheet**  
  - Type: Google Drive node.  
  - Role: Checks if a Google Sheet exists for logging.  
  - OnError: Continue.  
  - Output: Passes to Sheet Exists? node.

- **Sheet Exists?**  
  - Type: If node.  
  - Role: Conditional branch based on sheet presence.  
  - True: Pass to Set Folder ID sheet.  
  - False: Pass to Create New Sheet.

- **Set Folder ID sheet**  
  - Type: Code node.  
  - Role: Stores or sets the ID for the Google Sheet for later use.  
  - Input: Sheet info.  
  - Output: Merges with other data.

- **Set Folder ID drive folder**  
  - Type: Code node.  
  - Role: Stores or sets the ID for the Drive folder for images.  
  - Input: Folder info.  
  - Output: Merges with other data.

- **Create Folder for images**  
  - Type: Google Drive node.  
  - Role: Creates a new folder in Google Drive if none exists.  
  - Output: Leads to Set Folder ID drive folder.

- **Create New Sheet**  
  - Type: Google Drive node.  
  - Role: Creates a new Google Sheet if none exists.  
  - Output: Leads to Set Folder ID sheet.

**Edge cases / failure types:**  
- Authentication errors with Google APIs.  
- Non-existent folders or sheets leading to creation steps.  
- API rate limits.  
- Date parsing errors in code nodes.

---

#### 1.2 Prompt Processing & AI Image Generation

**Overview:**  
Splits prompts into batches, generates images using the Flux1-schnell model, and manages asynchronous task timing.

**Nodes Involved:**  
- Google Sheets1  
- Create Loop Indexes  
- Set Topic  
- Prompt Generator  
- Split Prompts  
- Merge Batches  
- Google Sheets2  
- OpenAI  
- 20 seconds  
- Generate Image  
- Wait  
- Get Images  
- Check if it has data?  
- Split Out  

**Node Details:**

- **Google Sheets1**  
  - Reads batched prompt data from Sheets.

- **Create Loop Indexes**  
  - Function node creating indexes for looped batch processing.

- **Set Topic**  
  - Sets the main topic or context for prompt generation.

- **Prompt Generator**  
  - Langchain chainLlm node generating refined prompts for image generation from input topics.

- **Split Prompts**  
  - Function node splitting prompts into manageable batches.

- **Merge Batches**  
  - Merges batched prompt data for downstream processing.

- **Google Sheets2**  
  - Stores batch info or interim data in Sheets.

- **OpenAI**  
  - Chat OpenAI node, possibly for text prompt enhancement or intermediate processing.

- **20 seconds**  
  - Wait node enforcing a delay of 20 seconds between image generation requests to comply with rate limits or task processing.

- **Generate Image**  
  - HTTP Request node calling the Flux1-schnell model API to generate images.  
  - Key parameters: prompt text, image size, batch size, negative prompts to exclude unwanted features.

- **Wait**  
  - Additional wait node to ensure asynchronous task completion before retrieving images.

- **Get Images**  
  - HTTP Request node to fetch generated images after task completion.

- **Check if it has data?**  
  - If node verifying if image data is available for further processing.  
  - True path: Continues to image download.  
  - False path: Wait and retry.

- **Split Out**  
  - Splits batched image data for parallel processing.

**Edge cases / failure types:**  
- API request failures or timeouts with Flux1-schnell.  
- Rate limiting, handled by wait nodes.  
- Empty or invalid prompt batches.  
- Delays causing timeouts or lost data.

---

#### 1.3 Image Downloading & Processing

**Overview:**  
Downloads generated images, performs composition and resizing, then analyzes image content to extract metadata.

**Nodes Involved:**  
- Download Images  
- Comp Images  
- Resize Image X2  
- Merge  
- Analyze images  
- Split Out data  
- Parse OpenAI Response  

**Node Details:**

- **Download Images**  
  - HTTP Request node downloading images from URLs received from AI generation.  
  - Handles multiple images in batch.

- **Comp Images**  
  - Edit Image node for compositing or enhancing images.

- **Resize Image X2**  
  - Edit Image node resizing images to twice the dimension or resolution.

- **Merge**  
  - Merges processed images for downstream tasks.

- **Analyze images**  
  - Langchain OpenAI node analyzing images to generate descriptive metadata including titles, categories, and keywords.  
  - Input: image data or references.

- **Split Out data**  
  - Splits JSON metadata array for individual handling.

- **Parse OpenAI Response**  
  - Code node parsing the JSON metadata response into structured fields for Sheets.

**Edge cases / failure types:**  
- Image download failures or corrupt files.  
- Image processing errors (composition or resizing).  
- OpenAI API failures or malformed JSON responses.  
- Parsing errors due to unexpected JSON format.

---

#### 1.4 Metadata Parsing & Catalog Updating

**Overview:**  
Processes extracted metadata and updates Google Sheets with image titles, keywords, and categories.

**Nodes Involved:**  
- Numbering  
- Google Sheets3  
- Edit Fields  
- Generate Image (reused)  
- Google Sheets4  
- Telegram  

**Node Details:**

- **Numbering**  
  - Code node adding sequence numbers or indexing metadata entries.

- **Google Sheets3**  
  - Writes detailed metadata (Title, Category, Filename, Keywords) into a Google Sheet for cataloging.

- **Edit Fields**  
  - Set node editing or preparing fields for image generation or metadata logging.

- **Generate Image**  
  - (Reused) Starts image generation after metadata processing.

- **Google Sheets4**  
  - Updates the original prompt sheet indicating images have been created (e.g., "Images created: YES").

- **Telegram**  
  - Telegram node sending notifications or confirmations about image generation and upload status.

**Edge cases / failure types:**  
- Sheet write failures or permission errors.  
- Telegram API errors or message delivery failures.  
- Data mismatches causing incorrect sheet updates.

---

#### 1.5 Google Drive Folder Management & Upload

**Overview:**  
Ensures proper folder structure in Google Drive exists and uploads generated images for storage and sharing.

**Nodes Involved:**  
- Upload Images  
- Merge2  
- Set Folder ID sheet  
- Set Folder ID drive folder  
- Create Folder for images  
- Create New Sheet  

**Node Details:**

- **Upload Images**  
  - Google Drive node uploading images into the designated folder.

- **Merge2**  
  - Merges data from folder and sheet ID setting nodes to keep state consistent.

- **Set Folder ID sheet** / **Set Folder ID drive folder**  
  - Code nodes maintaining folder and sheet IDs for reuse.

- **Create Folder for images** / **Create New Sheet**  
  - Create nodes as fallback if folders or sheets do not exist.

**Edge cases / failure types:**  
- Upload failures due to network or permission issues.  
- Folder or sheet missing causing upload failures.  
- Conflicts in file naming or folder structure.

---

#### 1.6 Notifications & Error Handling

**Overview:**  
Captures workflow errors and sends notifications via Telegram, logging error details into a Google Sheet.

**Nodes Involved:**  
- Error Trigger  
- Log Error  
- Telegram1  

**Node Details:**

- **Error Trigger**  
  - Trigger node activating on workflow errors.

- **Log Error**  
  - Google Sheets node logging error details into a dedicated sheet for monitoring.

- **Telegram1**  
  - Telegram node sending error alerts to configured recipients.

**Edge cases / failure types:**  
- Missing error details due to node failure.  
- Telegram message failures.  
- Sheet write permission issues.

---

### 3. Summary Table

| Node Name               | Node Type                       | Functional Role                             | Input Node(s)                    | Output Node(s)                | Sticky Note                                   |
|-------------------------|--------------------------------|--------------------------------------------|---------------------------------|------------------------------|-----------------------------------------------|
| Schedule Trigger        | scheduleTrigger                 | Initiates workflow on schedule              | -                               | Set Date Info                |                                               |
| Set Date Info           | code                           | Sets date info for filtering                 | Schedule Trigger                | filter data date             |                                               |
| filter data date        | code                           | Filters data by date                         | Set Date Info                  | Google Sheets                |                                               |
| Google Sheets           | googleSheets                   | Reads prompts from sheet                     | filter data date               | Select Prompt                |                                               |
| Select Prompt           | code                           | Selects and prepares prompt                  | Google Sheets                  | Check the folder, Check sheet|                                               |
| Check the folder        | googleDrive                    | Checks for existing Drive folder             | Select Prompt                 | Folder Exists?               |                                               |
| Folder Exists?          | if                             | Branches on folder existence                  | Check the folder               | Set Folder ID drive folder / Create Folder for images |                                               |
| Check sheet             | googleDrive                    | Checks for existing Google Sheet              | Select Prompt                 | Sheet Exists?                |                                               |
| Sheet Exists?           | if                             | Branches on sheet existence                   | Check sheet                   | Set Folder ID sheet / Create New Sheet |                                               |
| Set Folder ID sheet     | code                           | Stores sheet ID                               | Sheet Exists?                 | Merge2                      |                                               |
| Set Folder ID drive folder | code                        | Stores folder ID                              | Folder Exists?                | Merge2                      |                                               |
| Create Folder for images| googleDrive                    | Creates folder if missing                      | Folder Exists? (false)         | Set Folder ID drive folder   |                                               |
| Create New Sheet        | googleDrive                    | Creates sheet if missing                       | Sheet Exists? (false)          | Set Folder ID sheet          |                                               |
| Google Sheets1          | googleSheets                   | Reads batch prompts                           | Schedule Trigger1              | Create Loop Indexes          |                                               |
| Create Loop Indexes     | function                       | Creates indexes for batch looping             | Google Sheets1                | Set Topic                   |                                               |
| Set Topic               | set                            | Sets prompt topic                             | Create Loop Indexes            | Prompt Generator            |                                               |
| Prompt Generator        | chainLlm (Langchain)           | Generates/refines prompts                      | Set Topic                    | Split Prompts               |                                               |
| Split Prompts           | function                       | Splits prompts into batches                    | Prompt Generator             | Merge Batches               |                                               |
| Merge Batches           | merge                          | Merges batch data                              | Split Prompts                | Google Sheets2              |                                               |
| Google Sheets2          | googleSheets                   | Writes batch data                              | Merge Batches                |                             |                                               |
| OpenAI                  | lmChatOpenAi (Langchain)       | Enhances prompt or intermediate text           | Prompt Generator             | Prompt Generator (ai_languageModel) |                                               |
| 20 seconds              | wait                           | Enforces delay between generation requests    | Generate Image               | Get Images                  |                                               |
| Generate Image          | httpRequest                    | Calls Flux1-schnell AI image generation API   | Edit Fields                 | 20 seconds                 |                                               |
| Wait                    | wait                           | Ensures task completion before fetching images | Check if it has data? (false) | Get Images                  |                                               |
| Get Images              | httpRequest                    | Retrieves generated images                      | 20 seconds                  | Check if it has data?        |                                               |
| Check if it has data?   | if                             | Checks if images are ready                      | Get Images                  | Split Out / Wait             |                                               |
| Split Out               | splitOut                      | Splits image batch for processing               | Check if it has data? (true)  | Download Images             |                                               |
| Download Images         | httpRequest                    | Downloads images from URLs                       | Split Out                   | Comp Images, Resize Image X2 |                                               |
| Comp Images             | editImage                     | Composites/enhances images                       | Download Images              | Analyze images              |                                               |
| Resize Image X2         | editImage                     | Resizes images                                  | Download Images              | Merge                      |                                               |
| Merge                   | merge                          | Merges image processing outputs                  | Resize Image X2, Comp Images | Code4                      |                                               |
| Code4                   | code                           | Prepares data for upload                          | Merge                       | Upload Images               |                                               |
| Upload Images           | googleDrive                    | Uploads images to Drive                           | Code4                       | Google Sheets4              |                                               |
| Google Sheets3          | googleSheets                   | Writes metadata (titles, keywords)                | Parse OpenAI Response       |                             |                                               |
| Analyze images          | openAi                        | Extracts metadata from images                      | Comp Images                 | Split Out data              |                                               |
| Split Out data          | splitOut                      | Splits metadata array into individual items         | Analyze images              | Parse OpenAI Response       |                                               |
| Parse OpenAI Response   | code                           | Parses metadata JSON into structured fields          | Split Out data              | Numbering, Google Sheets3   |                                               |
| Numbering               | code                           | Adds numbering/indexing to metadata entries          | Parse OpenAI Response       | Merge                      |                                               |
| Google Sheets4          | googleSheets                   | Updates sheet to mark images created                 | Upload Images               | Telegram                   |                                               |
| Telegram                | telegram                      | Sends notifications about workflow progress          | Google Sheets4              |                             |                                               |
| Error Trigger           | errorTrigger                  | Captures errors                                    | -                           | Log Error, Telegram1        |                                               |
| Log Error               | googleSheets                   | Logs error details                                 | Error Trigger              |                             |                                               |
| Telegram1               | telegram                      | Sends error notifications                          | Error Trigger              |                             |                                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node** to initiate the workflow on desired intervals (e.g., daily).

2. **Add a Code node "Set Date Info"** to calculate or set the date range for filtering prompts.

3. **Add a Code node "filter data date"** to filter prompts from the Google Sheet based on the date info.

4. **Add a Google Sheets node "Google Sheets"** configured to read prompts from your designated spreadsheet.

5. **Add a Code node "Select Prompt"** to process and prepare prompts for generation.

6. **Add two Google Drive nodes: "Check the folder" and "Check sheet"** to verify existence of image storage folder and metadata sheet. Configure with correct Drive credentials.

7. **Add two If nodes: "Folder Exists?" and "Sheet Exists?"** to branch logic based on presence.

8. **Add Google Drive nodes "Create Folder for images" and "Create New Sheet"** to create missing Drive folder and Google Sheet respectively.

9. **Add Code nodes "Set Folder ID sheet" and "Set Folder ID drive folder"** to store IDs for use in uploads and logging.

10. **Connect nodes to ensure folder and sheet IDs are merged and available downstream.**

11. **Add a Google Sheets node "Google Sheets1"** to read batch prompts for generation.

12. **Add a Function node "Create Loop Indexes"** to generate batch indexes for prompt processing.

13. **Add a Set node "Set Topic"** to assign topics for prompt generation.

14. **Add a Langchain Chain LLM node "Prompt Generator"** to generate/refine prompts using OpenAI or similar.

15. **Add a Function node "Split Prompts"** to divide prompts into batches.

16. **Add a Merge node "Merge Batches"** to combine batch data.

17. **Add a Google Sheets node "Google Sheets2"** to write batch info back to Sheets.

18. **Add an OpenAI Chat node "OpenAI"** for prompt enhancement or intermediate processing if needed.

19. **Add an HTTP Request node "Generate Image"** configured to send requests to Flux1-schnell API for image generation. Include parameters such as prompt text, width, height, batch size, guidance scale, and negative prompts.

20. **Add a Wait node "20 seconds"** to respect rate limits between generation calls.

21. **Add an HTTP Request node "Get Images"** to retrieve generated images after task completion.

22. **Add an If node "Check if it has data?"** to verify availability of generated images.

23. **Add a Wait node "Wait"** to retry image fetching if no data is available.

24. **Add a Split Out node "Split Out"** to split image batch data.

25. **Add an HTTP Request node "Download Images"** to download images from their URLs.

26. **Add Edit Image nodes "Comp Images" and "Resize Image X2"** to process images (composition and resizing).

27. **Add a Merge node "Merge"** to combine processed images.

28. **Add a Code node "Code4"** to prepare data for upload.

29. **Add a Google Drive node "Upload Images"** configured to upload images to the Drive folder.

30. **Add a Langchain OpenAI node "Analyze images"** to generate metadata such as titles, categories, and keywords based on image content.

31. **Add a Split Out node "Split Out data"** to split metadata array.

32. **Add a Code node "Parse OpenAI Response"** to parse JSON metadata into structured fields.

33. **Add a Code node "Numbering"** to index metadata entries.

34. **Add a Google Sheets node "Google Sheets3"** to write parsed metadata into a Google Sheet.

35. **Add a Google Sheets node "Google Sheets4"** to update the original prompt sheet marking images as created.

36. **Add a Telegram node "Telegram"** to send notifications on successful image creation.

37. **Add an Error Trigger node "Error Trigger"** to capture workflow errors.

38. **Add a Google Sheets node "Log Error"** to log errors into a dedicated sheet.

39. **Add a Telegram node "Telegram1"** to send error notifications.

40. **Connect all nodes according to the dependency and flow described in the Summary Table and Block-by-Block Analysis.**

41. **Configure credentials for Google Sheets, Google Drive, OpenAI (Langchain), and Telegram nodes.** Ensure proper OAuth2 and API keys with appropriate scopes.

42. **Test each section independently before enabling the full workflow.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                | Context or Link                                                                                      |
|-----------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| The workflow uses the Flux1-schnell AI model for image generation.                                                          | Model: "Qubico/flux1-schnell" (HTTP API)                                                           |
| Negative prompts are used to reduce image artifacts like blurry, low quality, distorted images, or unwanted objects.        | Included in Generate Image node parameters.                                                        |
| Google Sheets and Drive API scopes require permissions for reading, writing, and creating files/folders in user Drive.      | Ensure OAuth2 credentials have these scopes enabled.                                               |
| Telegram nodes send notifications and error alerts to configured chat IDs; a bot token with proper chat permissions is needed.| Prepare Telegram Bot and obtain chat ID for notifications.                                          |
| The Langchain OpenAI nodes require an OpenAI API key configured in the n8n credentials.                                       | Used for prompt generation and image metadata extraction.                                          |
| The workflow uses wait nodes to handle rate limiting and asynchronous processing of image generation tasks.                  | Wait times are currently set to 20 seconds after image generation calls.                            |
| Google Sheets logging includes updating image creation status ("YES"/"NO") and writing metadata (title, category, keywords).| Helps track progress and maintain a catalog of generated images.                                   |
| Error handling is centralized with an Error Trigger node logging to Sheets and sending Telegram alerts.                     | Facilitates monitoring and quick error response.                                                  |

---

**Disclaimer:**  
The text and workflow described originate exclusively from an automated process created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.