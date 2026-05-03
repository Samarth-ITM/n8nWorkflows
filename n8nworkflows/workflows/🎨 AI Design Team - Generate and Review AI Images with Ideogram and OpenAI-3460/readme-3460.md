🎨 AI Design Team - Generate and Review AI Images with Ideogram and OpenAI

https://n8nworkflows.xyz/workflows/---ai-design-team---generate-and-review-ai-images-with-ideogram-and-openai-3460


# 🎨 AI Design Team - Generate and Review AI Images with Ideogram and OpenAI

### 1. Workflow Overview

This workflow, titled **"🎨 AI Design Team - Generate and Review AI Images with Ideogram and OpenAI"**, is designed to automate the generation, review, and management of AI-created images for graphic design teams, creative agencies, marketing teams, or freelancers. It streamlines the creative process by integrating AI image generation, automated quality review, structured asset management, and notification delivery.

The workflow is logically divided into the following blocks:

- **1.1 Setup and Initialization**  
  Handles initial folder and file creation in Google Drive, uploads a CSV template, and sends setup details via email.

- **1.2 Input Reception and Creative Brief Preparation**  
  Receives user input via a form trigger and prepares the creative brief for image generation.

- **1.3 AI Image Generation (Ideogram Integration)**  
  Sends the creative brief to Ideogram API to generate images and processes the response.

- **1.4 Image Download and Automated Review (OpenAI Integration)**  
  Downloads generated images, sends them to OpenAI for quality evaluation, and parses the AI review output.

- **1.5 Conditional Workflow Branching Based on Review**  
  Routes images for approval, modification, or rejection based on AI recommendations.

- **1.6 Remix Generation and Asset Management**  
  For images requiring modification, generates remixed images, uploads them to Google Drive, and updates Google Sheets metadata.

- **1.7 Final Asset Compilation and Notification**  
  Compiles metadata into Google Sheets, uploads CSV files, and sends email notifications when images are ready.

---

### 2. Block-by-Block Analysis

#### 2.1 Setup and Initialization

**Overview:**  
This block runs once to create the necessary Google Drive folder structure and upload a CSV template. It also sends an email with setup details and folder links.

**Nodes Involved:**  
- When clicking ‘Test workflow’ (Manual Trigger)  
- Google Drive - Create Folder  
- Google Drive - Create Generations Folder  
- Spreadsheet (Set Node)  
- Convert to File  
- Google Drive - Upload Spreadsheet  
- Gmail - Send Setup Details  
- Sticky Note (Setup instructions and reminders)

**Node Details:**

- **When clicking ‘Test workflow’**  
  - Type: Manual Trigger  
  - Role: Starts the setup process manually.  
  - Inputs: None  
  - Outputs: Triggers Google Drive folder creation.  
  - Edge cases: None.

- **Google Drive - Create Folder**  
  - Type: Google Drive Folder Creation  
  - Role: Creates root folder "Graphic_Design_Team" in My Drive root.  
  - Config: Folder name fixed; parent folder is root.  
  - Inputs: Trigger from manual node.  
  - Outputs: Folder ID used by next node.  
  - Edge cases: Folder already exists or permission errors.

- **Google Drive - Create Generations Folder**  
  - Type: Google Drive Folder Creation  
  - Role: Creates subfolder "Image_Generations" inside the root folder created above.  
  - Config: Uses folder ID from previous node as parent.  
  - Inputs: Folder ID from previous node.  
  - Outputs: Folder ID for CSV upload.  
  - Edge cases: Permission issues, folder name conflicts.

- **Spreadsheet (Set Node)**  
  - Type: Set  
  - Role: Prepares CSV header string for the image metadata CSV file.  
  - Config: Sets a CSV header string in a variable `csvFile`.  
  - Inputs: Folder creation output.  
  - Outputs: CSV content for file creation.  
  - Edge cases: None.

- **Convert to File**  
  - Type: Convert to File  
  - Role: Converts CSV string to a file named `n8n-graphicdesignteam.csv`.  
  - Config: Binary property name set to "spreadsheet".  
  - Inputs: CSV string from Set node.  
  - Outputs: Binary file data for upload.  
  - Edge cases: Conversion failures.

- **Google Drive - Upload Spreadsheet**  
  - Type: Google Drive File Upload  
  - Role: Uploads the CSV file into the "Image_Generations" folder.  
  - Config: Uses folder ID from "Create Generations Folder".  
  - Inputs: Binary file from Convert to File node.  
  - Outputs: File metadata including webViewLink.  
  - Edge cases: Upload failures, permission errors.

- **Gmail - Send Setup Details**  
  - Type: Gmail Send  
  - Role: Sends an email with links to the created folders and CSV file, including folder IDs for configuration.  
  - Config: Email address must be updated by user; message includes dynamic links and IDs.  
  - Inputs: File and folder metadata from previous nodes.  
  - Outputs: None  
  - Edge cases: Authentication errors, email delivery failures.

- **Sticky Note (Setup instructions)**  
  - Type: Sticky Note  
  - Role: Provides detailed setup instructions and reminders for users.  
  - Content: Step-by-step setup guide and links.  
  - Inputs/Outputs: None.

---

#### 2.2 Input Reception and Creative Brief Preparation

**Overview:**  
Receives user inputs from a form submission and formats these inputs into a structured creative brief for image generation.

**Nodes Involved:**  
- On form submission (Form Trigger)  
- creative brief (Set Node)  
- Sticky Note (Form description and branding)

**Node Details:**

- **On form submission**  
  - Type: Form Trigger  
  - Role: Entry point for user input via a web form with fields for prompt, audience, aspect ratio, model, magic prompt option, style type, and negative prompt.  
  - Config: Form fields are required or optional as per design; includes dropdowns for controlled vocabularies.  
  - Inputs: User form data  
  - Outputs: JSON with user inputs for downstream nodes.  
  - Edge cases: Missing required fields, invalid inputs.

- **creative brief**  
  - Type: Set  
  - Role: Maps form inputs into a structured JSON object for the image request and stores folder IDs for Google Drive.  
  - Config: Assigns fields like prompt, model, aspect ratio, magic prompt, style type, negative prompt, and audience. Also includes a fixed GenerationsFolderId for Google Drive.  
  - Inputs: Form submission data  
  - Outputs: Structured creative brief JSON for Ideogram API.  
  - Edge cases: Missing or malformed inputs.

- **Sticky Note (Form description)**  
  - Type: Sticky Note  
  - Role: Provides branding and links for the form interface.  
  - Inputs/Outputs: None.

---

#### 2.3 AI Image Generation (Ideogram Integration)

**Overview:**  
Sends the creative brief to the Ideogram API to generate AI images and processes the response.

**Nodes Involved:**  
- Ideogram Image generator (HTTP Request)  
- SetImageData (Set Node)  
- GET image (HTTP Request)  
- Google Drive (Folder Upload)  
- Google Sheets (Metadata Append)  
- genImageURL1 (Set Node)  
- TheImageURL (Set Node)  
- Download Image (HTTP Request)

**Node Details:**

- **Ideogram Image generator**  
  - Type: HTTP Request  
  - Role: Sends POST request to Ideogram API `/generate` endpoint with image request JSON.  
  - Config: Uses HTTP Header Auth with Ideogram credentials; content-type JSON; accepts JSON response.  
  - Inputs: creative brief JSON  
  - Outputs: JSON response with image data including URL, seed, style type, resolution, and safety flag.  
  - Edge cases: API errors, authentication failures, rate limits, malformed requests.

- **SetImageData**  
  - Type: Set  
  - Role: Extracts and maps key image data from Ideogram response and creative brief into structured JSON fields for downstream use.  
  - Config: Assigns image URL, seed, NSFW flag, prompt, model, style type, aspect ratio, negative prompt, and magic prompt option.  
  - Inputs: Ideogram Image generator output  
  - Outputs: Structured image metadata JSON  
  - Edge cases: Missing fields in API response.

- **GET image**  
  - Type: HTTP Request  
  - Role: Downloads the generated image file from the URL provided by Ideogram.  
  - Config: Simple GET request to image URL.  
  - Inputs: Image URL from SetImageData  
  - Outputs: Binary image data  
  - Edge cases: Download failures, broken URLs.

- **Google Drive**  
  - Type: Google Drive Folder Upload  
  - Role: Uploads the generated image file to a timestamped folder inside the "Image_Generations" folder.  
  - Config: Folder name includes timestamp; parent folder ID from creative brief.  
  - Inputs: Binary image data from GET image  
  - Outputs: File metadata including webViewLink  
  - Edge cases: Upload failures, permission issues.

- **Google Sheets**  
  - Type: Google Sheets Append  
  - Role: Appends image metadata (prompt, seed, NSFW, dimensions, URLs, generation parameters) as a new row in a Google Sheet.  
  - Config: Maps fields from SetImageData and Google Drive output; uses a specific sheet and document ID.  
  - Inputs: Google Drive file metadata and SetImageData JSON  
  - Outputs: Confirmation of append operation  
  - Edge cases: Sheet access errors, schema mismatches.

- **genImageURL1**  
  - Type: Set  
  - Role: Sets a simplified JSON with the generated image URL for downstream nodes.  
  - Inputs: Google Sheets output  
  - Outputs: JSON with `genImage.url`  
  - Edge cases: None.

- **TheImageURL**  
  - Type: Set  
  - Role: Transfers the generated image URL into a field named `theimage.url` for download.  
  - Inputs: genImageURL1 output  
  - Outputs: JSON with `theimage.url`  
  - Edge cases: None.

- **Download Image**  
  - Type: HTTP Request  
  - Role: Downloads the image file from the URL for review.  
  - Config: Response format set to file, output binary property "image".  
  - Inputs: TheImageURL output  
  - Outputs: Binary image data for AI review  
  - Edge cases: Download failures.

---

#### 2.4 Image Download and Automated Review (OpenAI Integration)

**Overview:**  
Uses OpenAI GPT-4o model to evaluate the generated image for quality and relevance, producing a structured recommendation and enhanced prompt.

**Nodes Involved:**  
- Image Reviewer (LangChain LLM Chain)  
- Structured Output Parser1 (LangChain Output Parser)  
- OpenAI Chat Model1 (OpenAI LLM)  
- Switch1 (Conditional Routing)

**Node Details:**

- **Image Reviewer**  
  - Type: LangChain Chain LLM  
  - Role: Sends the downloaded image and creative brief to OpenAI for evaluation, requesting JSON output with recommendation, explanation, and enhanced prompt.  
  - Config: Custom prompt instructing AI to evaluate spelling, aesthetics, audience alignment; expects JSON output with fields: overall_recommendation, explanation, enhanced_image_prompt.  
  - Inputs: Binary image data and creative brief audience and prompt  
  - Outputs: AI JSON evaluation  
  - Edge cases: API timeouts, malformed AI response.

- **Structured Output Parser1**  
  - Type: LangChain Structured Output Parser  
  - Role: Parses the AI JSON output to extract structured fields for workflow use.  
  - Config: Defines schema with enums and strings for recommendation, explanation, and enhanced prompt.  
  - Inputs: Raw AI output from Image Reviewer  
  - Outputs: Parsed JSON fields  
  - Edge cases: Parsing errors if AI output is invalid JSON.

- **OpenAI Chat Model1**  
  - Type: LangChain OpenAI Chat Model  
  - Role: Provides the GPT-4o model for the Image Reviewer node.  
  - Config: Model set to "gpt-4o".  
  - Inputs: Prompt from Image Reviewer  
  - Outputs: AI response  
  - Edge cases: API key issues, rate limits.

- **Switch1**  
  - Type: Switch  
  - Role: Routes workflow based on AI recommendation: "Use as is" or others (modifications/reject).  
  - Config: Checks `overall_recommendation` field.  
  - Inputs: Parsed AI output  
  - Outputs: Two branches: approved images trigger email notification; others trigger remix process.  
  - Edge cases: Unexpected recommendation values.

---

#### 2.5 Conditional Workflow Branching Based on Review

**Overview:**  
Branches the workflow to either send notification for approved images or initiate remix generation for images needing modification.

**Nodes Involved:**  
- Switch1 (from previous block)  
- Gmail (Notification)  
- RE creative brief (Set Node for remix)  

**Node Details:**

- **Switch1**  
  - See above.

- **Gmail**  
  - Type: Gmail Send  
  - Role: Sends notification email that the image is ready for use.  
  - Config: Email address must be updated; subject and message are fixed.  
  - Inputs: Triggered when recommendation is "Use as is".  
  - Outputs: None  
  - Edge cases: Email delivery failures.

- **RE creative brief**  
  - Type: Set  
  - Role: Prepares a revised creative brief using the enhanced prompt from AI review for remixing the image.  
  - Config: Sets fixed values for aspect ratio, model, magic prompt, negative prompt, and image weight; copies enhanced prompt and explanation from AI output.  
  - Inputs: AI output from Structured Output Parser1  
  - Outputs: Remix creative brief JSON  
  - Edge cases: Missing enhanced prompt or explanation.

---

#### 2.6 Remix Generation and Asset Management

**Overview:**  
Generates a remixed image based on the enhanced prompt, uploads it to Google Drive, and updates Google Sheets with remix metadata.

**Nodes Involved:**  
- Download Image3 (HTTP Request)  
- ideogram Remix (HTTP Request)  
- Set Upload Fields1 (Set Node)  
- Get Remixed Image (HTTP Request)  
- Google Drive Remix Image (Google Drive Upload)  
- Google Sheets - Add Remix (Google Sheets Append)  
- genImageURL2 (Set Node)  
- TheImageURL (Set Node)  
- Download Image (HTTP Request) (reused for remix download)

**Node Details:**

- **Download Image3**  
  - Type: HTTP Request  
  - Role: Downloads the original image file for remixing.  
  - Config: Response format file, output binary "image".  
  - Inputs: Remix creative brief image URL.  
  - Outputs: Binary image data.  
  - Edge cases: Download failures.

- **ideogram Remix**  
  - Type: HTTP Request  
  - Role: Sends a multipart form-data POST request to Ideogram remix API with the remix creative brief and image file.  
  - Config: Uses HTTP Header Auth with Ideogram credentials; includes image_request JSON and binary image file.  
  - Inputs: Binary image and remix creative brief JSON.  
  - Outputs: Remix image generation response JSON.  
  - Edge cases: API errors, authentication failures.

- **Set Upload Fields1**  
  - Type: Set  
  - Role: Extracts remix image data and sets fields for upload and metadata.  
  - Config: Maps prompt, resolution, seed, style type, URL, resemblance weight, magic prompt option.  
  - Inputs: Remix API response and remix creative brief.  
  - Outputs: Structured remix image metadata.  
  - Edge cases: Missing fields.

- **Get Remixed Image**  
  - Type: HTTP Request  
  - Role: Downloads the remixed image file from the URL.  
  - Config: Simple GET request.  
  - Inputs: Remix image URL from Set Upload Fields1.  
  - Outputs: Binary image data.  
  - Edge cases: Download failures.

- **Google Drive Remix Image**  
  - Type: Google Drive Upload  
  - Role: Uploads the remixed image file to the "Image_Generations" folder with a seed and timestamped name.  
  - Config: Folder ID from creative brief.  
  - Inputs: Binary image data from Get Remixed Image.  
  - Outputs: File metadata including webViewLink.  
  - Edge cases: Upload failures.

- **Google Sheets - Add Remix**  
  - Type: Google Sheets Append  
  - Role: Appends remix image metadata to the Google Sheet.  
  - Config: Maps remix metadata fields including prompt, seed, NSFW flag, dimensions, generation parameters, and drive link.  
  - Inputs: Google Drive Remix Image metadata and Set Upload Fields1 data.  
  - Outputs: Append confirmation.  
  - Edge cases: Sheet access errors.

- **genImageURL2**  
  - Type: Set  
  - Role: Sets remix image URL for downstream use.  
  - Inputs: Google Sheets - Add Remix output.  
  - Outputs: JSON with remix image URL.  
  - Edge cases: None.

- **TheImageURL**  
  - Type: Set  
  - Role: Transfers remix image URL for download or further processing.  
  - Inputs: genImageURL2 output.  
  - Outputs: JSON with remix image URL.  
  - Edge cases: None.

- **Download Image** (reused)  
  - See above.

---

#### 2.7 Final Asset Compilation and Notification

**Overview:**  
Compiles all image metadata into Google Sheets, uploads CSV files, and sends email notifications when images are ready for creative use.

**Nodes Involved:**  
- Google Sheets (Append original images)  
- Google Sheets - Add Remix (Append remixed images)  
- Gmail (Notification for approved images)  
- Gmail - Send Setup Details (Setup email)

**Node Details:**

- **Google Sheets**  
  - See 2.3 Google Sheets node.

- **Google Sheets - Add Remix**  
  - See 2.6 Google Sheets - Add Remix node.

- **Gmail**  
  - See 2.5 Gmail node.

- **Gmail - Send Setup Details**  
  - See 2.1 Gmail - Send Setup Details node.

---

### 3. Summary Table

| Node Name                     | Node Type                      | Functional Role                                      | Input Node(s)                        | Output Node(s)                     | Sticky Note                                                                                          |
|-------------------------------|--------------------------------|-----------------------------------------------------|------------------------------------|----------------------------------|----------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’  | Manual Trigger                 | Starts setup process                                | None                               | Google Drive - Create Folder      | # Run Setup First **ONCE**                                                                          |
| Google Drive - Create Folder   | Google Drive Folder            | Creates root folder "Graphic_Design_Team"           | When clicking ‘Test workflow’      | Google Drive - Create Generations Folder |                                                                                                    |
| Google Drive - Create Generations Folder | Google Drive Folder    | Creates subfolder "Image_Generations"                | Google Drive - Create Folder       | Spreadsheet                      |                                                                                                    |
| Spreadsheet                   | Set                            | Prepares CSV header string                           | Google Drive - Create Generations Folder | Convert to File               |                                                                                                    |
| Convert to File               | Convert to File                | Converts CSV string to file                          | Spreadsheet                       | Google Drive - Upload Spreadsheet |                                                                                                    |
| Google Drive - Upload Spreadsheet | Google Drive File Upload    | Uploads CSV file to "Image_Generations" folder      | Convert to File                   | Gmail - Send Setup Details        |                                                                                                    |
| Gmail - Send Setup Details    | Gmail Send                    | Sends setup email with folder and file links        | Google Drive - Upload Spreadsheet | None                            |                                                                                                    |
| Sticky Note                  | Sticky Note                   | Setup instructions and reminders                     | None                             | None                            | # n8n Graphic Design Team \n## _Setup Instructions_\n\n... (detailed setup instructions)           |
| On form submission            | Form Trigger                  | Receives user input from form                         | None                             | creative brief                   |                                                                                                    |
| creative brief               | Set                            | Prepares structured creative brief JSON              | On form submission               | Ideogram Image generator         |                                                                                                    |
| Ideogram Image generator      | HTTP Request                  | Sends image generation request to Ideogram API      | creative brief                  | SetImageData                    |                                                                                                    |
| SetImageData                 | Set                            | Extracts and maps image data from Ideogram response  | Ideogram Image generator         | GET image                      |                                                                                                    |
| GET image                   | HTTP Request                  | Downloads generated image file                        | SetImageData                   | Google Drive                   |                                                                                                    |
| Google Drive                | Google Drive Upload           | Uploads generated image to Drive                      | GET image                     | Google Sheets                  |                                                                                                    |
| Google Sheets               | Google Sheets Append          | Appends image metadata to Google Sheet                | Google Drive                   | genImageURL1                   |                                                                                                    |
| genImageURL1                | Set                            | Sets generated image URL                              | Google Sheets                  | TheImageURL                   |                                                                                                    |
| TheImageURL                 | Set                            | Prepares image URL for download                       | genImageURL1                   | Download Image                |                                                                                                    |
| Download Image              | HTTP Request                  | Downloads image for AI review                         | TheImageURL                   | Image Reviewer                |                                                                                                    |
| Image Reviewer              | LangChain Chain LLM           | Sends image and brief to OpenAI for evaluation       | Download Image                 | Switch1                      |                                                                                                    |
| Structured Output Parser1   | LangChain Output Parser       | Parses AI JSON evaluation output                      | Image Reviewer                | Switch1                      |                                                                                                    |
| OpenAI Chat Model1          | LangChain OpenAI Model        | Provides GPT-4o model for AI evaluation               | Image Reviewer (AI input)       | Image Reviewer (AI output)     |                                                                                                    |
| Switch1                    | Switch                        | Routes workflow based on AI recommendation            | Structured Output Parser1       | Gmail / RE creative brief      |                                                                                                    |
| Gmail                      | Gmail Send                   | Sends notification email for approved images          | Switch1 (approved branch)       | None                         |                                                                                                    |
| RE creative brief          | Set                            | Prepares remix creative brief from AI enhanced prompt | Switch1 (modification branch)   | Download Image3               |                                                                                                    |
| Download Image3            | HTTP Request                  | Downloads original image for remixing                  | RE creative brief              | ideogram Remix               |                                                                                                    |
| ideogram Remix             | HTTP Request                  | Sends remix request to Ideogram API                    | Download Image3               | Set Upload Fields1           |                                                                                                    |
| Set Upload Fields1         | Set                            | Extracts remix image data for upload and metadata      | ideogram Remix                | Get Remixed Image            |                                                                                                    |
| Get Remixed Image          | HTTP Request                  | Downloads remixed image file                            | Set Upload Fields1            | Google Drive Remix Image     |                                                                                                    |
| Google Drive Remix Image   | Google Drive Upload           | Uploads remixed image to Drive                          | Get Remixed Image             | Google Sheets - Add Remix    |                                                                                                    |
| Google Sheets - Add Remix  | Google Sheets Append          | Appends remix image metadata to Google Sheet           | Google Drive Remix Image      | genImageURL2                 |                                                                                                    |
| genImageURL2               | Set                            | Sets remix image URL                                   | Google Sheets - Add Remix     | TheImageURL                 |                                                                                                    |
| TheImageURL                | Set                            | Prepares remix image URL for download                   | genImageURL2                  | Download Image              |                                                                                                    |
| Sticky Note2               | Sticky Note                   | Reminder to select spreadsheet from list               | None                         | None                        | ## Select Spreadsheet from List                                                                    |
| Sticky Note3               | Sticky Note                   | Displays branding image                                  | None                         | None                        | ![image](https://fillin8n.realsimple.dev/ideoGener8r_Team.png)                                     |
| Sticky Note4               | Sticky Note                   | Reminder to set email                                    | None                         | None                        | # Set Your Email                                                                                   |
| Sticky Note5               | Sticky Note                   | Reminder to set email                                    | None                         | None                        | # Set Your Email                                                                                   |
| Sticky Note6               | Sticky Note                   | Branding and credits                                     | None                         | None                        | ## Created by **[Real Simple Solutions](https://realsimple.dev)** More templates 👉 **[Click Here](https://n8n.io/creators/joeperes/)** |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Name: "When clicking ‘Test workflow’"  
   - Purpose: Manual start for setup.

2. **Create Google Drive Folder Node**  
   - Name: "Google Drive - Create Folder"  
   - Operation: Create folder named "Graphic_Design_Team" in root of "My Drive".  
   - Credentials: Google Drive OAuth2.

3. **Create Google Drive Folder Node**  
   - Name: "Google Drive - Create Generations Folder"  
   - Operation: Create folder named "Image_Generations" inside folder created above.  
   - Parent folder ID: Output ID from previous node.  
   - Credentials: Google Drive OAuth2.

4. **Create Set Node**  
   - Name: "Spreadsheet"  
   - Purpose: Define CSV header string for image metadata.  
   - Assign: `csvFile` with CSV header string:  
     `created_at,image,Link,prompt,timings,seed,nsfw,width,height,type,drive_link,GenModel,GenPrompt,GenAspectRatio,GenStyleType,GenNegativePrompt`

5. **Create Convert to File Node**  
   - Name: "Convert to File"  
   - Input: `csvFile` string from previous node.  
   - File name: `n8n-graphicdesignteam.csv`  
   - Output binary property: `spreadsheet`.

6. **Create Google Drive Upload Node**  
   - Name: "Google Drive - Upload Spreadsheet"  
   - Upload file: Binary data from Convert to File node.  
   - Folder ID: From "Google Drive - Create Generations Folder" node.  
   - Credentials: Google Drive OAuth2.

7. **Create Gmail Send Node**  
   - Name: "Gmail - Send Setup Details"  
   - Send to: Your email address (update before use).  
   - Subject: "n8n Graphic Design Team Setup - 🔴 Important Links"  
   - Message: Include links and folder IDs dynamically from previous nodes.  
   - Credentials: Gmail OAuth2.

8. **Create Form Trigger Node**  
   - Name: "On form submission"  
   - Configure form fields: prompt (string), audience (string), Aspect Ratio (dropdown), model (dropdown), magic prompt (dropdown), style type (dropdown), negative prompt (string).  
   - Webhook ID: Auto-generated.

9. **Create Set Node**  
   - Name: "creative brief"  
   - Map form inputs to structured JSON fields for image request and include Google Drive folder ID for generations folder.  
   - Fields: prompt, aspect_ratio, model, magic_prompt_option, style_type, negative_prompt, targetAudiance, setup.GenerationsFolderId.

10. **Create HTTP Request Node**  
    - Name: "Ideogram Image generator"  
    - Method: POST  
    - URL: `https://api.ideogram.ai/generate`  
    - Body: JSON with `image_request` from creative brief.  
    - Authentication: HTTP Header Auth with Ideogram API key.  
    - Headers: Accept and Content-Type as application/json.

11. **Create Set Node**  
    - Name: "SetImageData"  
    - Extract image data fields from Ideogram response and creative brief for downstream use.

12. **Create HTTP Request Node**  
    - Name: "GET image"  
    - Method: GET  
    - URL: Image URL from SetImageData.  
    - Purpose: Download generated image.

13. **Create Google Drive Upload Node**  
    - Name: "Google Drive"  
    - Upload image file to a timestamped folder inside "Image_Generations" folder.  
    - Credentials: Google Drive OAuth2.

14. **Create Google Sheets Append Node**  
    - Name: "Google Sheets"  
    - Append image metadata row to Google Sheet.  
    - Map fields from SetImageData and Google Drive output.  
    - Credentials: Google Sheets OAuth2.

15. **Create Set Node**  
    - Name: "genImageURL1"  
    - Set `genImage.url` from Google Sheets output.

16. **Create Set Node**  
    - Name: "TheImageURL"  
    - Set `theimage.url` from genImageURL1.

17. **Create HTTP Request Node**  
    - Name: "Download Image"  
    - Download image file from TheImageURL.  
    - Response format: file, output binary "image".

18. **Create LangChain Chain LLM Node**  
    - Name: "Image Reviewer"  
    - Model: OpenAI GPT-4o (configured in next node).  
    - Prompt: Custom prompt to evaluate image for spelling, aesthetics, audience alignment; expects JSON output with overall_recommendation, explanation, enhanced_image_prompt.  
    - Input: Binary image and creative brief audience and prompt.

19. **Create LangChain Output Parser Node**  
    - Name: "Structured Output Parser1"  
    - Schema: Defines expected JSON fields from AI output.

20. **Create LangChain OpenAI Chat Model Node**  
    - Name: "OpenAI Chat Model1"  
    - Model: GPT-4o  
    - Credentials: OpenAI API key.

21. **Connect OpenAI Chat Model1 to Image Reviewer (AI input)**  
    - Connect output of OpenAI Chat Model1 to AI input of Image Reviewer.

22. **Create Switch Node**  
    - Name: "Switch1"  
    - Condition: Check `overall_recommendation` equals "Use as is".  
    - Outputs: Branch 1 (approved), Branch 2 (modifications/reject).

23. **Create Gmail Send Node**  
    - Name: "Gmail"  
    - Sends notification email for approved images.  
    - Update email address before use.

24. **Create Set Node**  
    - Name: "RE creative brief"  
    - Prepares remix creative brief using enhanced prompt from AI output.  
    - Sets fixed parameters for remix generation.

25. **Create HTTP Request Node**  
    - Name: "Download Image3"  
    - Downloads original image for remixing.

26. **Create HTTP Request Node**  
    - Name: "ideogram Remix"  
    - Sends remix request to Ideogram API with multipart form-data including remix creative brief and image file.

27. **Create Set Node**  
    - Name: "Set Upload Fields1"  
    - Extracts remix image data for upload and metadata.

28. **Create HTTP Request Node**  
    - Name: "Get Remixed Image"  
    - Downloads remixed image file.

29. **Create Google Drive Upload Node**  
    - Name: "Google Drive Remix Image"  
    - Uploads remixed image to "Image_Generations" folder.

30. **Create Google Sheets Append Node**  
    - Name: "Google Sheets - Add Remix"  
    - Appends remix image metadata to Google Sheet.

31. **Create Set Node**  
    - Name: "genImageURL2"  
    - Sets remix image URL.

32. **Create Set Node**  
    - Name: "TheImageURL" (reuse)  
    - Prepares remix image URL for download.

33. **Connect nodes accordingly**  
    - Follow the connection logic as per the workflow JSON, ensuring outputs feed into correct inputs.

34. **Add Sticky Notes**  
    - Add sticky notes with setup instructions, branding, and reminders at appropriate positions.

35. **Configure Credentials**  
    - Set up and link credentials for Gmail OAuth2, Google Drive OAuth2, Google Sheets OAuth2, OpenAI API, and Ideogram API.

36. **Test Workflow**  
    - Run the manual trigger to create folders and CSV file.  
    - Check email for setup details.  
    - Import CSV into Google Sheets and update Google Sheets nodes with new sheet.  
    - Update folder IDs in creative brief node.  
    - Submit form to generate and review images.

---

### 5. General Notes & Resources

| Note Content                                                                                              | Context or Link                                                                                  |
|-----------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Workflow created by **Real Simple Solutions**. More templates available at [n8n Creator Joe Peres](https://n8n.io/creators/joeperes/) | Branding and credits                                                                            |
| Setup instructions and workflow overview image: ![image](https://fillin8n.realsimple.dev/ideoGener8r_Team.png) | Visual branding and workflow context                                                           |
| Form description includes links: Created by <a href="https://realsimple.dev">Real Simple Solutions</a> and template link <a href="https://n8n.io/creators/joeperes/">Click Here</a> | Form UI branding and external resource links                                                   |
| Ensure all credentials (Gmail, Google Drive, Google Sheets, OpenAI, Ideogram) are properly configured before running the workflow | Critical for successful API integrations                                                       |
| The workflow uses only standard n8n nodes and official integrations; no custom or community nodes required | Simplifies deployment and maintenance                                                          |
| The workflow requires n8n version supporting LangChain nodes (version 1.2+ for output parser, 1.4 for chain LLM) | Version-specific requirements for AI evaluation nodes                                         |

---

This document provides a comprehensive understanding of the workflow structure, node configurations, and logic flow, enabling advanced users or AI agents to reproduce, modify, and troubleshoot the workflow effectively.