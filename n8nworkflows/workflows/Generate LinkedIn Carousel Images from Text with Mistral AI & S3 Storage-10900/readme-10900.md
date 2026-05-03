Generate LinkedIn Carousel Images from Text with Mistral AI & S3 Storage

https://n8nworkflows.xyz/workflows/generate-linkedin-carousel-images-from-text-with-mistral-ai---s3-storage-10900


# Generate LinkedIn Carousel Images from Text with Mistral AI & S3 Storage

### 1. Workflow Overview

This n8n workflow automates the generation of LinkedIn carousel images from user-submitted text using Mistral AI for text processing and AWS S3 for image storage. It is designed for social media content creators or marketing teams aiming to create visually appealing carousel posts with minimal manual effort.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception and AI Processing:** Receives user text input via a chat trigger, processes and cleans it with an AI Agent powered by Mistral Cloud, producing structured JSON output with titles and subtexts for carousel slides.
- **1.2 Output Parsing and Normalization:** Validates AI output into a structured format and normalizes titles and subtexts, generating safe file names for images.
- **1.3 Image Template Retrieval:** Downloads predefined high-resolution carousel slide templates from Google Drive.
- **1.4 Image Editing:** Renders the AI-generated titles and subtexts onto the downloaded templates using multi-step editing operations.
- **1.5 Image Upload to S3:** Uploads the edited images to an AWS S3 bucket, ensuring unique and collision-free filenames.
- **1.6 Aggregation and Response Formatting:** Aggregates image URLs and formats a final response message containing markdown image embeds and downloadable links for all carousel images.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and AI Processing

- **Overview:**  
  This block receives user input via a chat interface and sends it to the Mistral AI model. The AI Agent cleans the input text and generates three sets of short titles and subtexts formatted as JSON for carousel creation.

- **Nodes Involved:**  
  - When chat message received  
  - Mistral Cloud Chat Model  
  - AI Agent  
  - Structured Output Parser

- **Node Details:**

  - **When chat message received**  
    - *Type & Role:* Chat trigger node; entry point for user input via a chat UI.  
    - *Configuration:* Public webhook with custom CSS for UI styling and initial greeting message ("hi paste your post").  
    - *Key Expressions:* None.  
    - *Connections:* Output → AI Agent.  
    - *Failures:* Webhook unavailability, user input errors.  
    - *Notes:* UI allows file/template selection (custom CSS).  
   
  - **Mistral Cloud Chat Model**  
    - *Type & Role:* Language model node using Mistral Cloud API.  
    - *Configuration:* Model set to "mistral-small-latest" with default options.  
    - *Credentials:* Mistral Cloud API key configured.  
    - *Connections:* Output → AI Agent (as AI language model input).  
    - *Failures:* Authentication errors, API rate limits, network timeouts.  
   
  - **AI Agent**  
    - *Type & Role:* LangChain AI agent node that processes and formats input text.  
    - *Configuration:*  
      - System message instructs to:  
        1. Clean input text by removing formatting issues but preserving emojis.  
        2. Create 3 carousel style texts (title + subtext pairs) with titles ≤5 words and subtexts 1–2 sentences.  
        3. Return output strictly as a JSON object.  
      - Example input/output included for validation.  
    - *Connections:* Main output → normalize title/name nodes.  
    - *Failures:* Non-JSON output breaking parser, timeout, or malformed output.  
   
  - **Structured Output Parser**  
    - *Type & Role:* Validates that AI Agent output conforms to expected JSON schema.  
    - *Configuration:* JSON schema expects keys: postclean, title1, subtext1, title2, subtext2, title3, subtext3.  
    - *Connections:* Output → AI Agent node as output parser input (in LangChain agent chain).  
    - *Failures:* Invalid JSON or schema mismatch causes failure in downstream processes.  
    - *Notes:* Critical for ensuring safe and predictable downstream data.

---

#### 2.2 Output Parsing and Normalization

- **Overview:**  
  This block extracts individual title and subtext pairs from the AI Agent output and generates safe filenames for images to be created later.

- **Nodes Involved:**  
  - normalize title,name 1  
  - normalize title,name 2  
  - normalize title,name 3  
  - normalize title,name 4

- **Node Details:**

  Each "normalize title,name X" node:  
  - *Type & Role:* Set node; assigns and formats variables for each slide.  
  - *Configuration:*  
    - Extracts respective titleX and subtextX from AI Agent JSON output fields.  
    - Creates a `safeName` string by taking the first word of the title, removing non-alphanumeric characters, appending the current date (YYYYMMDD), and a slide-specific number suffix (1 to 4).  
  - *Expressions:*  
    - e.g., `{{$json.output.title1}}` for title, `{{$json.output.subtext1}}` for subtext.  
    - `safeName`: built with regex replace and date slicing expressions.  
  - *Connections:* Each node outputs to a Google Drive image download node corresponding to a slide template.  
  - *Failures:* Missing or malformed AI output fields, empty titles causing filename issues.

---

#### 2.3 Image Template Retrieval

- **Overview:**  
  Downloads pre-designed slide templates from Google Drive to be used as backgrounds for carousel images.

- **Nodes Involved:**  
  - Google Drive get image 1  
  - Google Drive (get image template)  
  - Google Drive (get image template)3  
  - Google Drive (get image template)4

- **Node Details:**

  Each Google Drive node:  
  - *Type & Role:* Google Drive node configured for file download.  
  - *Configuration:*  
    - `fileId` parameter set by URL (to be replaced with actual template PNG URLs).  
    - Operation set to "download".  
  - *Credentials:* Google OAuth2 configured.  
  - *Connections:* Output → corresponding Edit Image node.  
  - *Failures:* Invalid file ID, permission issues, network errors.  
  - *Notes:* Templates are expected to be high resolution and consistent in aspect ratio for uniform carousel slides.

---

#### 2.4 Image Editing

- **Overview:**  
  Adds the AI-generated title and subtext text overlays onto each downloaded template image using multi-step image editing operations.

- **Nodes Involved:**  
  - Edit Image 1  
  - Edit Image2  
  - Edit Image3

- **Node Details:**

  Each Edit Image node:  
  - *Type & Role:* Edit Image node to render text over images.  
  - *Configuration:*  
    - Operation: multiStep.  
    - Steps include:  
      - Drawing title text with Arial Bold font, font size 45, color #F3EFEF, positioned near bottom (Y=550), with a max line length for wrapping.  
      - Drawing subtext with Arial regular font, font size 35, color #F3DFDF, positioned below title (Y=700).  
    - Text content expressions: `={{ $json.title }}` and `={{ $json.subtext }}` from input JSON.  
  - *Connections:* Output → corresponding S3 upload nodes.  
  - *Failures:* Font file not found, text overflow, long text causing layout issues.

---

#### 2.5 Image Upload to S3

- **Overview:**  
  Uploads the edited images to an AWS S3 bucket, using safe filenames to avoid collisions, making images accessible for downstream use.

- **Nodes Involved:**  
  - S32  
  - S3  
  - S35  
  - S34

- **Node Details:**

  Each S3 node:  
  - *Type & Role:* Amazon S3 upload node.  
  - *Configuration:*  
    - Operation: upload.  
    - Bucket name: placeholder `<bucketname>` to be replaced.  
    - File name: `={{ $json.safeName }}` for unique filenames per slide.  
  - *Credentials:* Configured AWS S3 credentials.  
  - *Connections:* Each outputs to a "get s3 url image X" Set node.  
  - *Failures:* Authentication issues, bucket permissions, filename conflicts.

---

#### 2.6 Aggregation and Response Formatting

- **Overview:**  
  Collects all uploaded image URLs, merges them, aggregates the data, and formats a final markdown message with image embeds and download links to be sent back to the user.

- **Nodes Involved:**  
  - get s3 url image 1  
  - get s3 url image 2  
  - get s3 url image 3  
  - get s3 url image 4  
  - Merge1  
  - Aggregate  
  - output format

- **Node Details:**

  - **get s3 url image X (Set nodes):**  
    - Assigns markdown image link strings using S3 URLs constructed with safe filenames.  
    - Expressions format markdown image embeds and downloadable link URLs.  
    - Outputs to Merge1 node.  
    - Failures: Incorrect URL templates or missing safeName.  

  - **Merge1 (Merge node):**  
    - Merges 4 inputs into one data array.  
    - Failures: Connectivity issues if inputs missing or delayed.  

  - **Aggregate (Aggregate node):**  
    - Aggregates all merged data for final processing.  
    - Failures: Data inconsistency or missing elements.  

  - **output format (Set node):**  
    - Formats a multi-line string message combining all images and download links using expressions pulling data from previous nodes.  
    - Outputs the final response message for delivery to the user.  
    - Failures: Expression errors if data missing or malformed.

---

### 3. Summary Table

| Node Name                     | Node Type                                | Functional Role                           | Input Node(s)                             | Output Node(s)                          | Sticky Note                                                                                      |
|-------------------------------|----------------------------------------|------------------------------------------|------------------------------------------|---------------------------------------|------------------------------------------------------------------------------------------------|
| When chat message received     | @n8n/n8n-nodes-langchain.chatTrigger  | Receives user text input via chat UI     | -                                        | AI Agent                              | Receives user post. Custom CSS configures UI and allows file/template selection.                |
| Mistral Cloud Chat Model       | @n8n/n8n-nodes-langchain.lmChatMistralCloud | Provides AI language model processing     | -                                        | AI Agent (languageModel input)         |                                                                                                |
| AI Agent                      | @n8n/n8n-nodes-langchain.agent         | Cleans input and generates 3 sets of titles/subtexts | When chat message received, Mistral Cloud Chat Model | normalize title,name 1,2,3,4          | Cleans input and returns 3 banner items as JSON: title1/subtext1, title2/subtext2, title3/subtext3. Titles ≤5 words. |
| Structured Output Parser       | @n8n/n8n-nodes-langchain.outputParserStructured | Validates AI JSON output                  | AI Agent (outputParser input)             | AI Agent (as output parser)            | Validates AI JSON. If parser fails, agent output is unsafe.                                    |
| normalize title,name 1         | n8n-nodes-base.set                     | Extracts and formats first title/subtext and safe filename | AI Agent                                  | Google Drive get image 1                |                                                                                                |
| normalize title,name 2         | n8n-nodes-base.set                     | Extracts and formats second title/subtext and safe filename | AI Agent                                  | Google Drive (get image template)      |                                                                                                |
| normalize title,name 3         | n8n-nodes-base.set                     | Extracts and formats third title/subtext and safe filename | AI Agent                                  | Google Drive (get image template)3     |                                                                                                |
| normalize title,name 4         | n8n-nodes-base.set                     | Generates safe filename for 4th slide    | AI Agent                                  | Google Drive (get image template)4     |                                                                                                |
| Google Drive get image 1       | n8n-nodes-base.googleDrive             | Downloads template image 1 from Google Drive | normalize title,name 1                     | Edit Image 1                          | Image Templates (Google Drive). Replace cached fileIds with your template PNGs.                 |
| Google Drive (get image template)  | n8n-nodes-base.googleDrive         | Downloads template image 2 from Google Drive | normalize title,name 2                     | Edit Image2                          |                                                                                                |
| Google Drive (get image template)3 | n8n-nodes-base.googleDrive          | Downloads template image 3 from Google Drive | normalize title,name 3                     | Edit Image3                          |                                                                                                |
| Google Drive (get image template)4 | n8n-nodes-base.googleDrive          | Downloads template image 4 from Google Drive | normalize title,name 4                     | S34                                 |                                                                                                |
| Edit Image 1                  | n8n-nodes-base.editImage               | Renders title and subtext on template 1 | Google Drive get image 1                   | S32                                 | Edit Image nodes render text on templates. Font size and position set per template.            |
| Edit Image2                   | n8n-nodes-base.editImage               | Renders title and subtext on template 2 | Google Drive (get image template)          | S3                                  |                                                                                                |
| Edit Image3                   | n8n-nodes-base.editImage               | Renders title and subtext on template 3 | Google Drive (get image template)3         | S35                                 |                                                                                                |
| S32                          | n8n-nodes-base.s3                      | Uploads edited image 1 to S3              | Edit Image 1                              | get s3 url image 1                   | S3 Uploads to bucket `<bucketname>`. Confirm public access or signed URLs.                      |
| S3                           | n8n-nodes-base.s3                      | Uploads edited image 2 to S3              | Edit Image2                              | get s3 url image 2                   |                                                                                                |
| S35                          | n8n-nodes-base.s3                      | Uploads edited image 3 to S3              | Edit Image3                              | get s3 url image 3                   |                                                                                                |
| S34                          | n8n-nodes-base.s3                      | Uploads edited image 4 to S3              | Google Drive (get image template)4        | get s3 url image 4                   |                                                                                                |
| get s3 url image 1            | n8n-nodes-base.set                     | Constructs markdown image link for image 1 | S32                                      | Merge1                              |                                                                                                |
| get s3 url image 2            | n8n-nodes-base.set                     | Constructs markdown image link for image 2 | S3                                       | Merge1                              |                                                                                                |
| get s3 url image 3            | n8n-nodes-base.set                     | Constructs markdown image link for image 3 | S35                                      | Merge1                              |                                                                                                |
| get s3 url image 4            | n8n-nodes-base.set                     | Constructs markdown image link for image 4 | S34                                      | Merge1                              |                                                                                                |
| Merge1                       | n8n-nodes-base.merge                   | Merges 4 image markdown link objects     | get s3 url image 1,2,3,4                  | Aggregate                          | Merge collects all uploaded images. Aggregate builds final message.                            |
| Aggregate                    | n8n-nodes-base.aggregate               | Aggregates merged image data into one    | Merge1                                   | output format                      |                                                                                                |
| output format                | n8n-nodes-base.set                     | Formats final markdown message with embeds and download links | Aggregate                                | -                                  |                                                                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Chat Trigger Node**  
   - Type: `@n8n/n8n-nodes-langchain.chatTrigger`  
   - Configure webhook as public with custom CSS for UI styling.  
   - Set initial message: "hi paste your post".  
   - This is the entry point for user input.

2. **Create Mistral Cloud Chat Model Node**  
   - Type: `@n8n/n8n-nodes-langchain.lmChatMistralCloud`  
   - Model: `mistral-small-latest`  
   - Credentials: Add Mistral Cloud API credentials.  
   - Connect output to AI Agent node (as AI language model input).

3. **Create Structured Output Parser Node**  
   - Type: `@n8n/n8n-nodes-langchain.outputParserStructured`  
   - JSON schema example: keys for `postclean`, `title1`, `subtext1`, `title2`, `subtext2`, `title3`, `subtext3`.  
   - Connect output parser input to AI Agent node.

4. **Create AI Agent Node**  
   - Type: `@n8n/n8n-nodes-langchain.agent`  
   - System Message:  
     - Clean input text (remove formatting issues except emojis).  
     - Create 3 carousel style title/subtext pairs, with titles max 5 words.  
     - Output only JSON object as per schema.  
   - Connect `main` input to Chat Trigger node output.  
   - Connect `ai_languageModel` input to Mistral Cloud Chat Model output.  
   - Connect `ai_outputParser` input to Structured Output Parser output.  
   - Connect output to "normalize title,name" nodes.

5. **Create Four "normalize title,name" Set Nodes**  
   - For each slide (1 to 4):  
     - Extract corresponding titleX and subtextX from AI Agent output.  
     - Create a `safeName` string: first word of title (cleaned), underscore, current date (YYYYMMDD), plus slide number suffix.  
   - Connect AI Agent output to each normalize node.

6. **Create Four Google Drive Nodes to Download Image Templates**  
   - Type: `n8n-nodes-base.googleDrive`  
   - Operation: download  
   - Credentials: Google Drive OAuth2 configured.  
   - File ID: Replace with your own template PNG URLs for 4 templates.  
   - Connect each normalize node output to corresponding Google Drive node.

7. **Create Four Edit Image Nodes**  
   - Type: `n8n-nodes-base.editImage`  
   - Operation: multiStep  
   - Steps:  
     - Add title text with Arial Bold, size 45, color #F3EFEF, Y=550, line length ~30-35 chars.  
     - Add subtext with Arial, size 35, color #F3DFDF, Y=700, line length ~60 chars.  
   - Text configured with expressions `={{ $json.title }}` and `={{ $json.subtext }}`.  
   - Connect Google Drive node outputs to corresponding Edit Image node inputs.

8. **Create Four S3 Upload Nodes**  
   - Type: `n8n-nodes-base.s3`  
   - Operation: upload  
   - Bucket Name: Set your S3 bucket name.  
   - Credentials: AWS S3 credentials configured.  
   - File Name: `={{ $json.safeName }}` for each node.  
   - Connect each Edit Image node output to corresponding S3 node.

9. **Create Four Set Nodes to Construct S3 Image URLs**  
   - Type: `n8n-nodes-base.set`  
   - For each image, create markdown embed string and download link using S3 URL and safeName.  
   - Expressions build strings like:  
     - `=![image1](https:s3url/{{ $('normalize title,name 1').item.json.safeName }})`  
     - Download link with query param to trigger download.  
   - Connect each S3 upload node output to corresponding Set node.

10. **Create Merge Node**  
    - Type: `n8n-nodes-base.merge`  
    - Number of inputs: 4  
    - Connect all four "get s3 url image" Set nodes outputs as inputs.

11. **Create Aggregate Node**  
    - Type: `n8n-nodes-base.aggregate`  
    - Aggregate all merged data into a single item.  
    - Connect Merge node output to Aggregate node.

12. **Create Final Set Node for Output Formatting**  
    - Type: `n8n-nodes-base.set`  
    - Compose multi-line message with all image embeds and download links using expressions that extract URLs from aggregated data.  
    - Connect Aggregate output to this node.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                    | Context or Link                                                                                       |
|-------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| This workflow turns user input text into a 3–4 slide LinkedIn carousel with AI-generated titles and subtexts rendered on image templates.      | Overview sticky note within workflow.                                                              |
| Replace all placeholder `<bucketname>` with your actual AWS S3 bucket name and confirm bucket policies for public read or signed URLs.        | S3 Upload sticky note.                                                                             |
| Replace Google Drive file IDs/URLs with your own high-resolution PNG templates. Templates should have consistent aspect ratios for best results.| Image Templates sticky note.                                                                       |
| Test the workflow initially with the chat UI to verify AI output correctness and image rendering. Titles must be ≤5 words; subtexts 1–2 sentences.| Setup instructions in overview sticky note.                                                       |
| Edit Image nodes use fonts located in `/usr/share/fonts/truetype/msttcorefonts/`. Confirm these fonts exist or adjust paths for your system.  | Edit Image sticky note.                                                                            |
| The final message includes markdown image embeds and clickable download links for user convenience.                                             | Merge & Aggregate sticky note.                                                                     |
| UI customization including color scheme and window size is done via CSS in the chat trigger node.                                               | Chat Trigger sticky note.                                                                          |
| Examples of image templates are linked within sticky notes as images hosted on Supabase.                                                        | Sticky notes with example template images.                                                        |

---

This documentation enables understanding, reproduction, and maintenance of the workflow, ensuring clarity of each processing step and integration requirement.