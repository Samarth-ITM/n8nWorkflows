Generate Educational Social Media Carousels with GPT-4.1, Templated.io & Google Drive

https://n8nworkflows.xyz/workflows/generate-educational-social-media-carousels-with-gpt-4-1--templated-io---google-drive-9654


# Generate Educational Social Media Carousels with GPT-4.1, Templated.io & Google Drive

### 1. Workflow Overview

This workflow automates the generation and storage of educational social media carousel posts using AI-driven content creation and templated graphic rendering. It targets content creators, social media managers, and educators who want to efficiently produce visually appealing, informative Instagram carousels on chosen topics.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Captures user input prompt via a secure form.
- **1.2 AI Content Generation:** Uses OpenAI GPT-4.1 to generate structured JSON content including titles, subtitles, topics, and visual suggestions.
- **1.3 Content Formatting:** Parses and cleans the AI-generated JSON for downstream use.
- **1.4 Folder Creation:** Creates a dedicated Google Drive folder named after the carousel title to store assets.
- **1.5 Image Retrieval:** Searches Pixabay for a cover image based on AI’s visual suggestion and extracts the first suitable result.
- **1.6 Carousel Rendering:** Uses Templated.io to generate PNG images of carousel slides by injecting content and images.
- **1.7 Asset Download and Storage:** Downloads the rendered images and uploads them to the newly created Google Drive folder.
- **1.8 Logging:** Records metadata about the created carousel in a Google Sheets spreadsheet for tracking and management.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  Collects a text prompt from the user via a secured form interface to initiate the carousel generation process.

- **Nodes Involved:**  
  - Form

- **Node Details:**  
  - **Form**  
    - Type: Form Trigger  
    - Role: Receives user input securely via a web form with Basic Authentication.  
    - Configuration: Single required textarea field labeled "Prompt". Form titled "Templated Form". Authenticated using Basic Auth credentials.  
    - Input: User-submitted prompt string  
    - Output: JSON object containing the prompt under key `Prompt`  
    - Edge Cases: Authentication failure, missing prompt input, form webhook errors.

#### 2.2 AI Content Generation

- **Overview:**  
  Calls OpenAI GPT-4.1 with a detailed system prompt plus the user prompt to generate structured JSON content for the carousel slides.

- **Nodes Involved:**  
  - Generate Content

- **Node Details:**  
  - **Generate Content**  
    - Type: OpenAI (LangChain node)  
    - Role: Submit prompt and system instructions to GPT-4.1 model to generate a JSON object with slide titles, subtitles, topic, description, and visual suggestions.  
    - Configuration: Uses a comprehensive system prompt that defines role, audience, style, tone, structure (slides 1-6), content rules, and output format. The user prompt is injected dynamically.  
    - Input: Prompt from Form node  
    - Output: Raw GPT response containing JSON text in the `message.content` field  
    - Edge Cases: API rate limits, malformed AI response, JSON parsing errors, credential/authentication issues.

#### 2.3 Content Formatting

- **Overview:**  
  Parses the raw JSON string from the AI response into a clean JavaScript object for use by subsequent nodes.

- **Nodes Involved:**  
  - Format content

- **Node Details:**  
  - **Format content**  
    - Type: Code (JavaScript)  
    - Role: Extract and parse the JSON string in the AI’s response under `message.content` → `.response`.  
    - Configuration: Uses `JSON.parse` on the AI message content to convert to JSON object with keys like `title-1`, `subtitle-1`, `topic`, `visual_suggestion`, `description`.  
    - Input: AI node output (raw text)  
    - Output: Parsed JSON object with expected keys  
    - Edge Cases: JSON parse failure if AI response is malformed or incomplete.

#### 2.4 Folder Creation

- **Overview:**  
  Creates a new Google Drive folder inside an existing "RRSS" folder, named after the carousel's first title.

- **Nodes Involved:**  
  - Create folder

- **Node Details:**  
  - **Create folder**  
    - Type: Google Drive  
    - Role: Create a subfolder in a fixed parent folder (RRSS) named after the carousel title (`title-1`).  
    - Configuration: Drive set to "My Drive", parent folder identified by fixed ID, folder name dynamically set from `title-1` field.  
    - Input: Parsed content from Format content node  
    - Output: Metadata about created folder including ID and webViewLink  
    - Edge Cases: Permission errors, folder name conflicts, API quota limits, invalid folder IDs.

#### 2.5 Image Retrieval

- **Overview:**  
  Searches Pixabay for images matching the AI's suggested visual keywords and extracts the first photo result.

- **Nodes Involved:**  
  - Get cover image  
  - Get first result

- **Node Details:**  
  - **Get cover image**  
    - Type: HTTP Request  
    - Role: Query Pixabay API for photos matching the `visual_suggestion` keyword.  
    - Configuration: Uses Pixabay API key, queries for photo type, horizontal orientation, 3 results per page.  
    - Input: Visual suggestion from formatted content  
    - Output: JSON response from Pixabay API with an array of image hits  
    - Edge Cases: API key expiry, network errors, zero results returned.

  - **Get first result**  
    - Type: Code (JavaScript)  
    - Role: Extract the first element from the Pixabay hits array and expose its fields (e.g., `largeImageURL`).  
    - Configuration: Returns the first hit object unmodified.  
    - Input: Pixabay API response  
    - Output: Single image metadata JSON object  
    - Edge Cases: Empty hits array causing undefined reference errors.

#### 2.6 Carousel Rendering

- **Overview:**  
  Generates carousel slide images using Templated.io by injecting textual content and the retrieved cover image URL into a predefined template.

- **Nodes Involved:**  
  - Create Renders

- **Node Details:**  
  - **Create Renders**  
    - Type: Templated.io node  
    - Role: Send slide titles, subtitles, topic, and the cover image URL to Templated.io to produce PNG images for each slide.  
    - Configuration:  
      - Format: PNG  
      - Template ID: fixed known template for Carousel  
      - Layers: Each slide’s title/subtitle fields bound to template text layers by name. The cover image URL set in the image layer `img-1`.  
    - Input: Parsed content and cover image metadata  
    - Output: JSON with URLs to rendered images  
    - Edge Cases: Template ID invalid, network/API errors, missing layer data, image URL invalid.

#### 2.7 Asset Download and Storage

- **Overview:**  
  Downloads the rendered carousel images and uploads them to the newly created Google Drive folder.

- **Nodes Involved:**  
  - Download renders  
  - Upload renders to Google Drive

- **Node Details:**  
  - **Download renders**  
    - Type: HTTP Request  
    - Role: Download binary image files from URLs returned by Templated.io.  
    - Configuration: URL set dynamically from previous node output.  
    - Input: Render URLs from Create Renders node  
    - Output: Binary file data  
    - Edge Cases: Download failures, timeouts, invalid URLs.

  - **Upload renders to Google Drive**  
    - Type: Google Drive  
    - Role: Upload downloaded PNG files into the folder created earlier.  
    - Configuration: File name set as `{{page}}.png` dynamically (page presumably refers to slide number), target folder ID from Create folder node.  
    - Input: Binary image data from Download renders node  
    - Output: Metadata about uploaded files  
    - Edge Cases: Permission errors, quota limits, invalid folder IDs.

#### 2.8 Logging

- **Overview:**  
  Logs metadata about the generated carousel to a Google Sheets spreadsheet for tracking purposes.

- **Nodes Involved:**  
  - Save in DB

- **Node Details:**  
  - **Save in DB**  
    - Type: Google Sheets  
    - Role: Append a new row with title, topic, creation timestamp, folder URL, description, and status.  
    - Configuration:  
      - Document and sheet IDs fixed  
      - Columns mapped explicitly: Title, Topic, Status ("Created"), Created At (formatted date), Folder URL, Description  
      - Operation: Append row  
      - Execute only once per workflow run  
    - Input: Output metadata from Create folder and formatted content nodes  
    - Output: Confirmation of row append  
    - Edge Cases: Sheet access permissions, API limits, formatting errors.

---

### 3. Summary Table

| Node Name                  | Node Type                   | Functional Role                                 | Input Node(s)              | Output Node(s)                         | Sticky Note                                                                                   |
|----------------------------|-----------------------------|------------------------------------------------|----------------------------|---------------------------------------|-----------------------------------------------------------------------------------------------|
| Form                       | Form Trigger                | Collects user prompt with basic auth           | -                          | Generate Content                      | ## Form Node The Form node collects a single “Prompt” (with basic auth) to kick off a new carousel request. |
| Generate Content            | OpenAI (LangChain)          | Generates structured JSON content via GPT-4.1 | Form                       | Format content                       | ## OpenAI (OpenAI GPT-4.1) uses your system prompt + the form’s Prompt to produce JSON with titles/subtitles, topic, description, and visual_suggestion. |
| Format content             | Code (JavaScript)            | Parses AI JSON response to clean object        | Generate Content            | Create folder                       | ## Format Content Format content parses the OpenAI raw message JSON and returns a clean object under the expected keys (e.g., title-1, subtitle-1, topic, visual_suggestion, description). |
| Create folder              | Google Drive                | Creates a subfolder inside RRSS named after title-1 | Format content             | Get cover image                    | ## Create Folder Create folder (Google Drive) creates a subfolder inside RRSS named with title-1 to store this carousel’s assets. |
| Get cover image            | HTTP Request                | Queries Pixabay API for images using visual suggestion | Create folder             | Get first result                  | ## Get Cover Image Get cover image (HTTP Request → Pixabay) searches for a photo using the parsed visual_suggestion as the keyword query. |
| Get first result           | Code (JavaScript)           | Extracts first Pixabay hit from API response   | Get cover image             | Create Renders                    | ## Get first result Get first result extracts the first Pixabay hit and exposes its fields (e.g., largeImageURL) for downstream use. |
| Create Renders             | Templated.io                | Renders carousel images using content & image | Get first result            | Download renders                  | ## Create Renders Create Renders (Templated.io) renders the design by injecting all slide titles/subtitles/topic and sets the image layer (img-1) to the Pixabay largeImageURL. |
| Download renders           | HTTP Request                | Downloads rendered images from Templated.io    | Create Renders              | Upload renders to Google Drive     | ## Download Renders Download renders (HTTP Request) downloads the rendered image(s) from the URL returned by Templated.io. |
| Upload renders to Google Drive | Google Drive            | Uploads downloaded images to created Drive folder | Download renders          | Save in DB                       | ## Upload renders to GD Upload renders to Google Drive uploads the downloaded binary file to the newly created Drive folder with the name {{page}}.png. |
| Save in DB                 | Google Sheets               | Logs carousel metadata into Google Sheets       | Upload renders to Google Drive | -                               | ## Save in DB Save in DB (Google Sheets) appends a new row logging timestamp, title, topic, Drive folder link, description, and a default Status = Created. |
| Sticky Note                | Sticky Note                 | Workflow overview note                          | -                          | -                                 | ## Generate and save Carousels for social media Using: - Form - OpenAI - Pixabay - Templated.io - Google Drive - Google Sheets |
| Sticky Note1               | Sticky Note                 | Form node explanation                           | -                          | -                                 | ## Form Node The Form node collects a single “Prompt” (with basic auth) to kick off a new carousel request. |
| Sticky Note2               | Sticky Note                 | OpenAI node explanation                         | -                          | -                                 | ## OpenAI (OpenAI GPT-4.1) uses your system prompt + the form’s Prompt to produce JSON with titles/subtitles, topic, description, and visual_suggestion. |
| Sticky Note3               | Sticky Note                 | Format content explanation                      | -                          | -                                 | ## Format Content Format content parses the OpenAI raw message JSON and returns a clean object under the expected keys (e.g., title-1, subtitle-1, topic, visual_suggestion, description). |
| Sticky Note4               | Sticky Note                 | Create folder explanation                        | -                          | -                                 | ## Create Folder Create folder (Google Drive) creates a subfolder inside RRSS named with title-1 to store this carousel’s assets. |
| Sticky Note5               | Sticky Note                 | Get first result explanation                     | -                          | -                                 | ## Get first result Get first result extracts the first Pixabay hit and exposes its fields (e.g., largeImageURL) for downstream use. |
| Sticky Note6               | Sticky Note                 | Get cover image explanation                      | -                          | -                                 | ## Get Cover Image Get cover image (HTTP Request → Pixabay) searches for a photo using the parsed visual_suggestion as the keyword query. |
| Sticky Note7               | Sticky Note                 | Create renders explanation                        | -                          | -                                 | ## Create Renders Create Renders (Templated.io) renders the design by injecting all slide titles/subtitles/topic and sets the image layer (img-1) to the Pixabay largeImageURL. |
| Sticky Note8               | Sticky Note                 | Download renders explanation                      | -                          | -                                 | ## Download Renders Download renders (HTTP Request) downloads the rendered image(s) from the URL returned by Templated.io. |
| Sticky Note9               | Sticky Note                 | Upload renders explanation                        | -                          | -                                 | ## Upload renders to GD Upload renders to Google Drive uploads the downloaded binary file to the newly created Drive folder with the name {{page}}.png. |
| Sticky Note10              | Sticky Note                 | Save in DB explanation                           | -                          | -                                 | ## Save in DB Save in DB (Google Sheets) appends a new row logging timestamp, title, topic, Drive folder link, description, and a default Status = Created. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Form Trigger Node ("Form")**  
   - Type: Form Trigger  
   - Configure a form titled "Templated Form" with one required textarea field labeled "Prompt".  
   - Enable Basic Authentication and configure credentials accordingly.  
   - Note the webhook URL generated for external access.

2. **Add OpenAI GPT Node ("Generate Content")**  
   - Type: OpenAI (LangChain)  
   - Set model to GPT-4.1.  
   - Use the detailed system prompt specifying role, audience, style, structure, content rules, and output format as JSON.  
   - Inject the user prompt dynamically from the Form node’s output.  
   - Configure OpenAI API credentials.

3. **Add Code Node ("Format content")**  
   - Type: Code (JavaScript)  
   - Use code to parse AI response:  
     ```js
     const json = JSON.parse($input.first().json.message.content).response;
     return json;
     ```  
   - Connect input from Generate Content node.

4. **Add Google Drive Node ("Create folder")**  
   - Type: Google Drive  
   - Operation: Create Folder  
   - Parent Folder ID: Set to the fixed RRSS folder ID.  
   - Folder Name: Use expression `={{ $json['title-1'] }}` from Format content node.  
   - Set Google Drive OAuth2 credentials.

5. **Add HTTP Request Node ("Get cover image")**  
   - Type: HTTP Request  
   - Method: GET  
   - URL: `https://pixabay.com/api/`  
   - Query Parameters:  
     - key = Pixabay API key  
     - q = Expression: `={{ $('Format content').item.json.visual_suggestion }}`  
     - image_type = "photo"  
     - orientation = "horizontal"  
     - per_page = 3  
   - Connect input from Create folder node.

6. **Add Code Node ("Get first result")**  
   - Type: Code (JavaScript)  
   - Code:  
     ```js
     const first = $input.first().json.hits[0];
     return first;
     ```  
   - Connect input from Get cover image node.

7. **Add Templated.io Node ("Create Renders")**  
   - Type: Templated.io  
   - Set format to PNG.  
   - Use the known Carousel template ID.  
   - Map layers with expressions to inject titles, subtitles, topic, and image URL from Format content and Get first result nodes.  
   - Set Templated.io API credentials.  
   - Connect input from Get first result node.

8. **Add HTTP Request Node ("Download renders")**  
   - Type: HTTP Request  
   - Method: GET  
   - URL: Use expression `={{ $json.url }}` from Create Renders node output.  
   - Connect input from Create Renders node.

9. **Add Google Drive Node ("Upload renders to Google Drive")**  
   - Type: Google Drive  
   - Operation: Upload File  
   - Folder ID: Expression `={{ $('Create folder').item.json.id }}`  
   - File Name: Expression `={{ $json.page }}.png` (page corresponds to slide number)  
   - Input Data Field Name: `data` (binary data from Download renders)  
   - Set Google Drive OAuth2 credentials.  
   - Connect input from Download renders node.

10. **Add Google Sheets Node ("Save in DB")**  
    - Type: Google Sheets  
    - Operation: Append Row  
    - Document ID and Sheet Name set to fixed spreadsheet and sheet.  
    - Map columns:  
      - Title = `={{ $('Create folder').item.json.name }}`  
      - Topic = `={{ $('Format content').item.json.topic }}`  
      - Status = "Created" (static)  
      - Created At = `={{ new Date($json.createdTime).toLocaleString() }}`  
      - Folder URL = `={{ $('Create folder').item.json.webViewLink }}`  
      - Description = `={{ $('Format content').item.json.description }}`  
    - Set Google Sheets OAuth2 credentials.  
    - Connect input from Upload renders to Google Drive node.  
    - Set node to execute only once per workflow run.

---

### 5. General Notes & Resources

| Note Content                                                                                                  | Context or Link                                  |
|---------------------------------------------------------------------------------------------------------------|-------------------------------------------------|
| Workflow integrates OpenAI's GPT-4.1 with Templated.io and Google Drive to automate social media carousel creation. | Project description                              |
| Pixabay API key used for image searches must be valid and have sufficient quota.                              | Pixabay API documentation                        |
| Google OAuth2 credentials are required for both Drive and Sheets nodes with relevant scopes enabled.          | Google Cloud Console OAuth2 setup                |
| Templated.io template ID "58756536-fd45-4e15-82aa-9189f0e042e9" must exist and be configured with the expected layers. | Templated.io dashboard                           |
| Form node uses Basic Auth to restrict access; credentials must be securely stored in n8n.                      | n8n form trigger documentation                   |
| Output JSON format expected from AI is strict; any change in system prompt or AI output may break parsing.     | OpenAI best practices                             |
| Logging into Google Sheets facilitates tracking and auditing of generated carousels with metadata.             | Google Sheets API documentation                   |

---

**Disclaimer:** The provided text is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.