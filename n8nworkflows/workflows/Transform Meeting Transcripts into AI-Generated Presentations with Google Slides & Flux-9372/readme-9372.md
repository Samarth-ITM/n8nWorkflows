Transform Meeting Transcripts into AI-Generated Presentations with Google Slides & Flux

https://n8nworkflows.xyz/workflows/transform-meeting-transcripts-into-ai-generated-presentations-with-google-slides---flux-9372


# Transform Meeting Transcripts into AI-Generated Presentations with Google Slides & Flux

### 1. Workflow Overview

This workflow automates the transformation of meeting transcripts into fully AI-generated sales presentations using Google Slides, Google Docs, Google Sheets, and AI models (OpenAI GPT-4 and Google Gemini). It is designed for sales, consulting, marketing, and product teams to convert client meeting notes into structured, branded presentation decks with rich visual illustrations.

The workflow is logically divided into four main blocks:

- **1.1 Create Presentation & Initialize Database**  
  Captures client input via form, creates a new Google Slides presentation by copying a template, and logs initial metadata into Google Sheets to track workflow progress.

- **1.2 Generate Presentation Plan (Text Content)**  
  Retrieves the meeting transcript, uses AI to analyze and generate a detailed structured presentation plan, saves it as a Google Doc, and updates the tracking database.

- **1.3 Generate Illustrations & Image Prompts**  
  Uses AI to create image prompts for presentation slides 3 through 8, calls an image generation API (OpenRouter nanobanana) to generate illustrations, uploads images to Google Drive, and updates the database with image status.

- **1.4 Edit Slides with Text and Images**  
  Retrieves slides and presentation plan, formats and updates all slide text placeholders, downloads AI-generated images from Drive, uploads images to ImgBB (to obtain direct URLs compatible with Google Slides), replaces placeholder images in the presentation, and finalizes the presentation status.

Each block is coordinated through Google Sheets as a central database and uses wait nodes to manage pacing and data consistency.

---

### 2. Block-by-Block Analysis

#### 1.1 Create Presentation & Initialize Database

**Overview:**  
This block triggers on form submission capturing client details and transcript URL, copies a Google Slides template to create a new presentation, and appends an entry to a Google Sheets database to track the presentation's lifecycle.

**Nodes Involved:**  
- On form submission  
- Create Presentation  
- Append in Main Sheet  
- Append in Images Sheet  
- Append in Client Sheet  
- Wait (delay)

**Node Details:**

- **On form submission**  
  - *Type:* Form Trigger  
  - *Role:* Entry point; captures "Client Name" (required) and "Meeting transcript URL" from user input form.  
  - *Config:* Webhook with form named "Nimbus Soft Presentation AI".  
  - *Outputs:* Triggers the creation of a new presentation.

- **Create Presentation**  
  - *Type:* HTTP Request (Google Drive API)  
  - *Role:* Copies a fixed Google Slides template to create a new presentation named "[Client Name] Sales Presentation" in a specified Drive folder.  
  - *Config:* POST to Drive API `/files/{template_id}/copy`, with JSON body specifying new file name and parent folder ID. Uses Google Drive OAuth2 credentials.  
  - *Failure modes:* Auth failures, invalid template ID, quota limits.

- **Append in Main Sheet**  
  - *Type:* Google Sheets Append/Update  
  - *Role:* Logs presentation metadata - status "PPT Plan Pending", description (presentation name), IDs, URLs, timestamp, and transcript URL into a master "Presentation Details" sheet.  
  - *Config:* Append or update mode by "Presentation ID". Uses Google Sheets OAuth2 credentials.  
  - *Potential issues:* Sheet access, schema mismatch.

- **Append in Images Sheet**  
  - *Type:* Google Sheets Append/Update  
  - *Role:* Initializes image generation status for the presentation as "Pending" in "Images & Illustrations" sheet.  
  - *Config:* Append or update by "Presentation ID".

- **Append in Client Sheet**  
  - *Type:* Google Sheets Append/Update  
  - *Role:* Initializes client-specific data placeholder in "Client Details" sheet.  
  - *Config:* Append or update by "Presentation ID".

- **Wait**  
  - *Type:* Wait node (1 minute delay)  
  - *Role:* Provides buffer time to ensure data consistency before next steps.

---

#### 1.2 Generate Presentation Plan (Text Content)

**Overview:**  
Retrieves the meeting transcript document, uses an AI agent to generate a structured presentation plan, saves the plan as a Google Doc, and updates the main database with the plan link and status.

**Nodes Involved:**  
- Get Pending Row  
- Get Transcript  
- Generate Presentation Plan (LangChain Agent)  
- Create PPT Plan Doc  
- Update PPT Plan Doc  
- Update row in Main Sheet + Trigger Image Gen  
- Wait1 (1 minute delay)

**Node Details:**

- **Get Pending Row**  
  - *Type:* Google Sheets Read  
  - *Role:* Finds the first presentation with status "PPT Plan Pending" to process.  
  - *Config:* Filter on "Status" column.

- **Get Transcript**  
  - *Type:* Google Docs Get  
  - *Role:* Fetches the meeting transcript document content using the URL from the sheet row.  
  - *Config:* Uses Google Docs OAuth2 credentials.

- **Generate Presentation Plan**  
  - *Type:* LangChain AI Agent  
  - *Role:* Uses GPT-4 or Google Gemini to analyze the transcript text and generate a detailed slide-by-slide presentation plan with structure, objectives, bullet points, and notes referencing NimbusSoft’s business profile.  
  - *Config:* System prompt contains detailed instructions on slide content, objectives, formatting, company profile, and expected output structure.  
  - *Failure modes:* AI model timeout, prompt errors, incomplete output.

- **Create PPT Plan Doc**  
  - *Type:* Google Docs Create  
  - *Role:* Creates a new Google Doc in a folder to store the generated presentation plan text.  
  - *Config:* Title matches the presentation description with "Plan" suffix.

- **Update PPT Plan Doc**  
  - *Type:* Google Docs Update  
  - *Role:* Inserts the generated presentation plan text into the newly created Google Doc.

- **Update row in Main Sheet + Trigger Image Gen**  
  - *Type:* Google Sheets Update  
  - *Role:* Updates the presentation status to "Images Pending", adds a link to the plan doc in the database, and triggers the next block for image generation.  
  - *Config:* Matches by "Presentation ID".

- **Wait1**  
  - *Type:* Wait node (1 minute delay)  
  - *Role:* Allows time for the update to propagate before image generation.

---

#### 1.3 Generate Illustrations & Image Prompts

**Overview:**  
AI agent generates structured image prompts based on the presentation plan for slides 3 to 8. Calls OpenRouter’s nanobanana model to generate images synchronously, converts the base64 outputs into files, uploads these images to Google Drive, and updates Google Sheets with image generation status.

**Nodes Involved:**  
- Get row(s) in sheet (Images Pending)  
- Get a document (Plan Doc)  
- Illustrations & Image Prompt Agent (AI Agent)  
- HTTP Request (OpenRouter API calls x6)  
- Edit Fields & Convert to File (x6)  
- Upload file (Google Drive x6)  
- Update row in sheet (Images & Illustrations x6)  
- Wait3 (1 minute delay)

**Node Details:**

- **Get row(s) in sheet**  
  - *Type:* Google Sheets Lookup  
  - *Role:* Finds presentations with status "Images Pending" to generate images.

- **Get a document**  
  - *Type:* Google Docs Get  
  - *Role:* Retrieves full presentation plan content.

- **Illustrations & Image Prompt Agent**  
  - *Type:* LangChain AI Agent with Structured Output Parser  
  - *Role:* Generates exactly six image prompts for slides 3-8 from the plan content, formatted as JSON.  
  - *Config:* System message instructs AI to ignore slides 1, 2, and 9 and produce concise descriptive text only without punctuation that could interfere with image generation.

- **HTTP Request (OpenRouter API calls)**  
  - *Type:* HTTP Request POST  
  - *Role:* Calls OpenRouter’s nanobanana model with the image prompts to generate images synchronously in base64 format. Authorization header with API key required.  
  - *Failure modes:* API limits, auth errors, malformed prompts.

- **Edit Fields & Convert to File**  
  - *Type:* Set + Convert to File  
  - *Role:* Extracts base64 data, MIME type, and constructs binary files for upload.

- **Upload file (Google Drive)**  
  - *Type:* Google Drive Upload  
  - *Role:* Uploads each image file into a designated Google Drive folder for storage and accessibility.

- **Update row in sheet (Images & Illustrations)**  
  - *Type:* Google Sheets Update  
  - *Role:* Updates image URLs and generation status in the images sheet per presentation.

- **Wait3**  
  - *Type:* Wait node (1 minute delay)  
  - *Role:* Ensures data consistency before next block.

---

#### 1.4 Edit Slides with Text and Images

**Overview:**  
Retrieves pending presentations marked "Editing Pending," fetches Google Slides and Docs content, processes final text formatting with AI, updates slide text placeholders, downloads AI-generated images from Drive, converts them to base64 to upload to ImgBB (to get direct image URLs), replaces images in Slides via Google Slides API, and updates status to "Completed."

**Nodes Involved:**  
- Get row(s) in sheet2 (Editing Pending)  
- Get row(s) in sheet3 (Images & Illustrations)  
- Get a document1 (Plan Doc)  
- Get a presentation1 (Google Slides)  
- Final Text Content Formatting (AI Agent)  
- Code (JavaScript for organizing AI output)  
- Replace text in a presentation (Google Slides API)  
- Download file 1-6 (Google Drive)  
- Convert to Base64 (Extract from File) x6  
- Upload to Imgbb x6 (HTTP Request)  
- Replace Image x6 (HTTP Request to Google Slides API)  
- Merge & Update row Presentation Details  
- Update row in sheet6 (Presentation Details status update)  
- Wait2 (1 minute delay)  
- Get row(s) in sheet4 (Image Update Pending)  
- Get a presentation (Google Slides)  
- Get a document2 (Plan Doc)  
- Get row(s) in sheet5 (Images & Illustrations)  
- Update row in sheet7 (Final status "Completed")  
- Merge1

**Node Details:**

- **Get row(s) in sheet2**  
  - Retrieves presentations with "Editing Pending" status.

- **Get row(s) in sheet3**  
  - Fetches images & illustrations for the presentation.

- **Get a document1**  
  - Retrieves the detailed presentation plan Google Doc.

- **Get a presentation1**  
  - Retrieves Google Slides presentation data including slide Object IDs.

- **Final Text Content Formatting**  
  - AI agent refines and structures slide text content into JSON strictly following professional presentation standards.  
  - System message enforces bullet point limits and formatting rules.

- **Code**  
  - JavaScript node parses and organizes the AI JSON output for easy templating and replacement in Slides.

- **Replace text in a presentation**  
  - Uses Google Slides API to replace text content in slide placeholders by matching Object IDs with refined text from AI.

- **Download file 1-6**  
  - Downloads generated images from Google Drive for each slide.

- **Convert to Base64 x6**  
  - Converts downloaded images to base64 for upload to ImgBB.

- **Upload to Imgbb x6**  
  - Uploads base64 images to ImgBB to obtain direct image URLs (required because Google Drive URLs are not directly usable for image insertion in Slides).

- **Replace Image x6**  
  - Uses Google Slides batchUpdate API to replace image placeholders with direct ImgBB URLs.

- **Merge & Update row Presentation Details**  
  - Combines results and updates the "Presentation Details" sheet, marking status as "Completed."

- **Wait2**  
  - Ensures consistent processing timing.

- **Get row(s) in sheet4, Get a presentation, Get a document2, Get row(s) in sheet5**  
  - Additional data fetch nodes for final stage validation and updates.

- **Update row in sheet7**  
  - Final update marking the presentation as "Completed" in the main tracking sheet.

- **Merge1**  
  - Combines final update outputs.

---

### 3. Summary Table

| Node Name                         | Node Type                   | Functional Role                              | Input Node(s)                    | Output Node(s)                          | Sticky Note                                                                                                             |
|----------------------------------|-----------------------------|----------------------------------------------|---------------------------------|----------------------------------------|-------------------------------------------------------------------------------------------------------------------------|
| On form submission               | Form Trigger                | Entry point, captures client & transcript info | -                               | Create Presentation                    | - Starts with an n8n form trigger capturing: client name, transcript URL (Google Docs), and submission timestamp         |
| Create Presentation             | HTTP Request                | Copies presentation template to new file      | On form submission              | Append in Main Sheet                   | - Creates a new presentation by copying a pre-configured template via Google Slides API                                  |
| Append in Main Sheet             | Google Sheets Append/Update | Logs presentation metadata                      | Create Presentation             | Append in Images Sheet                 | - Saves presentation details and form inputs to Google Sheets database (central tracking hub)                             |
| Append in Images Sheet           | Google Sheets Append/Update | Initializes image generation status             | Append in Main Sheet            | Append in Client Sheet                 | - Automatically triggers the presentation plan generation workflow                                                       |
| Append in Client Sheet           | Google Sheets Append/Update | Initializes client-specific data                 | Append in Images Sheet          | Wait                                 |                                                                                                                         |
| Wait                           | Wait                       | Delay for data consistency                      | Append in Client Sheet          | Get Pending Row                       |                                                                                                                         |
| Get Pending Row                 | Google Sheets Read          | Finds presentation with status "PPT Plan Pending" | Wait                          | Get Transcript                       |                                                                                                                         |
| Get Transcript                 | Google Docs Get             | Retrieves transcript document content            | Get Pending Row                | Generate Presentation Plan            |                                                                                                                         |
| Generate Presentation Plan      | LangChain AI Agent          | Generates structured presentation plan            | Get Transcript                | Create PPT Plan Doc                  | - AI agent analyzes transcript to identify client pain points and solution fit                                           |
| Create PPT Plan Doc             | Google Docs Create          | Creates Google Doc to store presentation plan      | Generate Presentation Plan      | Update PPT Plan Doc                  |                                                                                                                         |
| Update PPT Plan Doc             | Google Docs Update          | Inserts generated plan text into Google Doc        | Create PPT Plan Doc             | Update row in Main Sheet + Trigger Image Gen |                                                                                                                         |
| Update row in Main Sheet + Trigger Image Gen | Google Sheets Update   | Updates status to "Images Pending" and plan link    | Update PPT Plan Doc             | Wait1                                |                                                                                                                         |
| Wait1                          | Wait                       | Delay for data consistency                      | Update row in Main Sheet + Trigger Image Gen | Get row(s) in sheet                   |                                                                                                                         |
| Get row(s) in sheet             | Google Sheets Read          | Finds presentation with "Images Pending" status    | Wait1                         | Get a document                      |                                                                                                                         |
| Get a document                 | Google Docs Get             | Retrieves presentation plan document content          | Get row(s) in sheet            | Illustrations & Image Prompt Agent     |                                                                                                                         |
| Illustrations & Image Prompt Agent | LangChain AI Agent          | Generates image prompts for slides 3 to 8            | Get a document                | HTTP Request (OpenRouter API calls) x6 | - AI agent creates specific illustration prompts based on the presentation plan                                         |
| HTTP Request / HTTP Request1-5 | HTTP Request                | Calls OpenRouter API to generate images              | Illustrations & Image Prompt Agent | Edit Fields x6                       | - Uses OpenRouter's nanobanana model to generate images (best for text accuracy in images)                               |
| Edit Fields / Edit Fields1-5    | Set                        | Extracts base64, MIME, filename from API response      | HTTP Request x6                 | Convert to File x6                   |                                                                                                                         |
| Convert to File / Convert to File1-5 | Convert to File             | Converts base64 image data to binary files             | Edit Fields x6                 | Upload file x6                      |                                                                                                                         |
| Upload file / Upload file1-5    | Google Drive Upload         | Uploads images to Google Drive                     | Convert to File x6              | Update row in sheet x6               |                                                                                                                         |
| Update row in sheet / Update row in sheet1-5 | Google Sheets Update       | Updates image URLs and generation statuses           | Upload file x6                 | Wait3                               |                                                                                                                         |
| Wait3                          | Wait                       | Delay for data consistency                      | Update row in sheet x6          | Get row(s) in sheet2                 |                                                                                                                         |
| Get row(s) in sheet2            | Google Sheets Read          | Finds presentations with status "Editing Pending"     | Wait3                         | Get row(s) in sheet3                |                                                                                                                         |
| Get row(s) in sheet3            | Google Sheets Read          | Retrieves images & illustrations for presentation      | Get row(s) in sheet2           | Get a document1                    |                                                                                                                         |
| Get a document1                | Google Docs Get             | Retrieves presentation plan document                    | Get row(s) in sheet3           | Get a presentation1                |                                                                                                                         |
| Get a presentation1            | Google Slides Get           | Retrieves Google Slides presentation data               | Get a document1               | Final Text Content Formatting       |                                                                                                                         |
| Final Text Content Formatting   | LangChain AI Agent          | Produces final polished slide text JSON                  | Get a presentation1           | Code                              | - AI agent generates final polished text for all slide components from the presentation plan                            |
| Code                          | JavaScript Code             | Organizes AI output JSON for slide text replacement       | Final Text Content Formatting | Replace text in a presentation       | - JavaScript function cleans and formats the output for Google Slides API                                                |
| Replace text in a presentation  | Google Slides API (batchUpdate) | Replaces all slide text placeholders with AI-generated content | Code                         | Update row in sheet6               | - Template must have unique \"Object IDs\" for each text element (titles, subtitles, body text)                          |
| Update row in sheet6            | Google Sheets Update        | Updates status to "Image Update Pending"                  | Replace text in a presentation | Wait2                             |                                                                                                                         |
| Wait2                          | Wait                       | Delay for data consistency                      | Update row in sheet6           | Get row(s) in sheet4               |                                                                                                                         |
| Get row(s) in sheet4            | Google Sheets Read          | Finds presentations with status "Image Update Pending"    | Wait2                         | Get a presentation                |                                                                                                                         |
| Get a presentation             | Google Slides Get           | Retrieves Google Slides presentation data               | Get row(s) in sheet4           | Get a document2                   |                                                                                                                         |
| Get a document2                | Google Docs Get             | Retrieves presentation plan document                    | Get row(s) in sheet4           | Get row(s) in sheet5             |                                                                                                                         |
| Get row(s) in sheet5            | Google Sheets Read          | Retrieves images & illustrations URLs                        | Get a document2               | Download file 1-6                |                                                                                                                         |
| Download file 1-6              | Google Drive Download       | Downloads AI-generated images from Drive                    | Get row(s) in sheet5           | Convert to Base64 1-6            |                                                                                                                         |
| Convert to Base64 1-6          | Extract from File           | Converts binary images to base64 for ImgBB upload           | Download file 1-6             | Upload to Imgbb 1-6             |                                                                                                                         |
| Upload to Imgbb 1-6            | HTTP Request                | Uploads base64 images to ImgBB to get direct image URLs      | Convert to Base64 1-6         | Replace Image 1-6               | - Uploads images to ImgBB to get direct .png URLs (Google Drive URLs don't work in automation)                          |
| Replace Image 1-6              | HTTP Request (Google Slides API) | Replaces slide images with ImgBB direct URLs                  | Upload to Imgbb 1-6           | Merge1                          | - Users need Free ImgBB API key (https://api.imgbb.com/) and Google Slides OAuth2 API credentials                       |
| Merge1                         | Merge                      | Combines image replacement results                         | Replace Image 1-6             | Update row in sheet7             |                                                                                                                         |
| Update row in sheet7            | Google Sheets Update        | Final update marking presentation status to "Completed"      | Merge1                        | -                                |                                                                                                                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Form Trigger Node:**  
   - Type: Form Trigger  
   - Configure with form title "Nimbus Soft Presentation AI"  
   - Add fields: "Client Name" (required), "Meeting transcript URL" (optional)  
   - Set webhook endpoint.

2. **Create HTTP Request Node (Google Drive):**  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://www.googleapis.com/drive/v3/files/{templateId}/copy` (replace `{templateId}` with your Google Slides template ID)  
   - Body (JSON):  
     ```json
     {
       "name": "{{ $json['Client Name'] }} Sales Presentation",
       "parents": ["folderId_for_presentations"]
     }
     ```  
   - Headers: `Content-Type: application/json`  
   - Authentication: Google Drive OAuth2 credentials

3. **Create Google Sheets Append/Update Nodes:**  
   - One to append presentation metadata to "Presentation Details" sheet  
   - One to append image generation status to "Images & Illustrations" sheet  
   - One to append client info to "Client Details" sheet  
   - Configure columns and matching keys accordingly  
   - Use Google Sheets OAuth2 credentials

4. **Add Wait Node:**  
   - Set delay 1 minute before proceeding

5. **Add Google Sheets Read Node:**  
   - Filter rows where status = "PPT Plan Pending" to process next presentation

6. **Add Google Docs Get Node:**  
   - Retrieve meeting transcript document content using URL from sheet data

7. **Add AI Agent Node (LangChain / OpenAI / Gemini):**  
   - Configure with system prompt describing NimbusSoft profile and detailed slide plan generation instructions  
   - Input: transcript content  
   - Output: structured presentation plan text

8. **Add Google Docs Create Node:**  
   - Create Google Doc titled "[Presentation Name] Plan" in designated folder

9. **Add Google Docs Update Node:**  
   - Insert AI-generated presentation plan into created Google Doc

10. **Add Google Sheets Update Node:**  
    - Update main sheet row: status = "Images Pending"  
    - Add link to plan doc

11. **Add Wait Node:**  
    - Delay 1 minute to allow data consistency

12. **Add Google Sheets Read Node:**  
    - Filter rows with status = "Images Pending"

13. **Add Google Docs Get Node:**  
    - Retrieve plan document content

14. **Add AI Agent Node (LangChain / OpenAI / Gemini):**  
    - System prompt instructs to generate exactly 6 image prompts for slides 3-8 in JSON format

15. **Add HTTP Request Nodes (x6):**  
    - POST to OpenRouter nanobanana API with each image prompt  
    - Include Authorization header with Bearer API key  
    - Receive base64 image data synchronously

16. **Add Set Nodes (x6):**  
    - Extract base64 data, MIME type, file extension for each image

17. **Add Convert to File Nodes (x6):**  
    - Convert base64 strings to binary files

18. **Add Google Drive Upload Nodes (x6):**  
    - Upload image files to configured folder in Google Drive

19. **Add Google Sheets Update Nodes (x6):**  
    - Update "Images & Illustrations" sheet with image URLs and mark generation status

20. **Add Wait Node:**  
    - Delay 1 minute before next processing

21. **Add Google Sheets Read Node:**  
    - Filter rows with status = "Editing Pending"

22. **Add Google Sheets Read Node:**  
    - Get corresponding images & illustrations rows

23. **Add Google Docs Get Node:**  
    - Get presentation plan document

24. **Add Google Slides Get Node:**  
    - Get presentation data including slide and object IDs

25. **Add AI Agent Node (LangChain/OpenAI/Gemini):**  
    - Refine and format final slide text content in JSON according to slide specs

26. **Add Code Node:**  
    - Parse and organize AI output JSON for templating

27. **Add Google Slides Replace Text Node:**  
    - Replace slide text placeholders by matching Object IDs with AI-generated text

28. **Add Google Sheets Update Node:**  
    - Update status to "Image Update Pending"

29. **Add Wait Node:**  
    - Delay 1 minute

30. **Add Google Sheets Read Node:**  
    - Filter rows with status = "Image Update Pending"

31. **Add Google Slides Get Node:**  
    - Retrieve presentation data

32. **Add Google Docs Get Node:**  
    - Retrieve plan doc

33. **Add Google Sheets Read Node:**  
    - Retrieve image URLs for the presentation

34. **Add Google Drive Download Nodes (x6):**  
    - Download images from Drive

35. **Add Extract From File Nodes (x6):**  
    - Convert binary images to base64 strings

36. **Add HTTP Request Nodes (x6):**  
    - Upload base64 images to ImgBB, retrieve direct image URLs  
    - Requires ImgBB API key

37. **Add HTTP Request Nodes (x6):**  
    - Use Google Slides batchUpdate API to replace images with ImgBB URLs  
    - Requires Google Slides OAuth2 credentials

38. **Add Merge Node:**  
    - Combine image replacement results

39. **Add Google Sheets Update Node:**  
    - Update final status to "Completed" in "Presentation Details" sheet

40. **Add Merge Node:**  
    - Combine final update outputs

41. **Add Sticky Notes:**  
    - Add explanatory sticky notes throughout workflow for documentation and clarity

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                   | Context or Link                                                                                                                                                    |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------|
| This automation is designed to be highly customizable: swap slide templates, update company profiles, change AI models (OpenAI/Gemini), or use different image generation APIs like DALL-E or Midjourney. Slides can be dynamically extended or modified based on content needs.                                                                                                    | Sticky Note 5                                                                                                                                                    |
| Full workflow documentation and best practices available in this detailed [Medium article](https://jmjomba.medium.com) by the creator Joseph. Connect via Twitter [@juppfy](https://x.com/juppfy) or email joseph@uppfy.com.                                                                                                                                                     | Sticky Note 5                                                                                                                                                    |
| Use cases include sales teams auto-generating pitch decks from discovery calls, consulting firms creating client proposals, marketing agencies building campaign presentations, product teams transforming user research, and training & education material generation.                                                                                                            | Sticky Note 5                                                                                                                                                    |
| Required API keys and credentials: OpenRouter API for nanobanana image generation, ImgBB API for free image hosting, OpenAI API for AI agents, Google Cloud Console for enabling Google Slides, Drive, Docs APIs, and Google AI Studio for Gemini API keys.                                                                                                                      | Sticky Note 5                                                                                                                                                    |
| Template presentation must have unique Object IDs for all text placeholders to enable accurate text replacement. JavaScript code is used to parse complex AI output JSON for effective text insertion into Google Slides. Google Drive URLs are not usable directly for image replacement, necessitating the use of ImgBB for direct image URLs. Users must secure all mentioned API keys. | Sticky Notes 9 & 10                                                                                                                                             |
| Example templates and sample meeting transcript provided (Google Docs and Sheets templates). It is recommended to make copies of all templates before use.                                                                                                                                                                                                                    | Sticky Note 5                                                                                                                                                    |
| The workflow uses wait nodes strategically to ensure data consistency and API rate limit handling.                                                                                                                                                                                                                                                                              | Overall workflow design                                                                                                                                          |

---

**Disclaimer:** The provided content is extracted from an automated n8n workflow and complies with all applicable content policies. All data processed is legal and publicly accessible.