Generate Viral Social Media Carousels with xAI Grok for Instagram, TikTok, LinkedIn & X

https://n8nworkflows.xyz/workflows/generate-viral-social-media-carousels-with-xai-grok-for-instagram--tiktok--linkedin---x-11024


# Generate Viral Social Media Carousels with xAI Grok for Instagram, TikTok, LinkedIn & X

### 1. Workflow Overview

This workflow automates the generation of viral social media carousels tailored for platforms like Instagram, TikTok, LinkedIn, and X (formerly Twitter). It is designed to take a user-defined theme and call-to-action (CTA), then use an AI agent to create structured slide content. Following this, it processes background templates, overlays text on images, and outputs finished carousel slides ready for posting.

The workflow is logically divided into four main blocks:

- **1.1 Input Reception and Parameter Setup:** Receives user input (theme, CTA, nickname) via webhook or manual trigger and prepares parameters.
- **1.2 AI Content Generation:** Uses an AI Agent powered by xAI Grok to generate a structured JSON containing seven carousel slides with titles and descriptions.
- **1.3 Content Parsing and Preparation:** Parses the AI output, sets style parameters, downloads background images, and splits slide data for batch processing.
- **1.4 Image Generation and Upload:** Edits images by overlaying titles and descriptions onto templates, then uploads final images to Google Drive for storage.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Parameter Setup

- **Overview:**  
This block initializes the workflow by capturing user inputs either via a webhook or manual trigger. It extracts essential fields such as the carousel theme, call-to-action text, and user nickname, setting them up for downstream processing.

- **Nodes Involved:**  
  - Webhook  
  - When clicking ‘Test workflow’ (Manual Trigger)  
  - Test Params (Code)  
  - Fields: Get Theme and CTA (Set)  
  - Sticky Note2 (Documentation)

- **Node Details:**

  - **Webhook**  
    - Type: HTTP Webhook Trigger  
    - Configuration: Listens for POST requests at path `/ai-carousel-generator`. Responds with the final output node’s response.  
    - Inputs: External HTTP requests  
    - Outputs: JSON body containing `theme`, `cta`, and `nickname` fields expected in the request body.  
    - Edge Cases: Missing or malformed payload, HTTP method mismatch.

  - **When clicking ‘Test workflow’**  
    - Type: Manual Trigger  
    - Configuration: Allows manual execution for testing.  
    - Inputs: None  
    - Outputs: Triggers `Test Params` node.

  - **Test Params**  
    - Type: Code (JavaScript)  
    - Configuration: Returns a sample JSON object emulating webhook input with default theme, nickname, and CTA.  
    - Key Expressions: Hardcoded values for testing purposes.  
    - Edge Cases: None (used only for testing).

  - **Fields: Get Theme and CTA**  
    - Type: Set  
    - Configuration: Extracts and assigns `theme`, `cta`, and `nickname` from the incoming JSON body to workflow variables.  
    - Key Expressions: `={{ $json.body.theme }}`, `={{ $json.body.cta }}`, `={{ $json.body.nickname }}`  
    - Inputs: From Webhook or Test Params nodes  
    - Outputs: Prepared JSON for AI Agent input.

  - **Sticky Note2**  
    - Type: Sticky Note (Documentation)  
    - Content: Describes this block as the workflow’s starting point handling key input parameters.

---

#### 2.2 AI Content Generation

- **Overview:**  
This block leverages an AI Agent using the xAI Grok language model to generate a structured JSON of seven carousel slides. Each slide contains a title and description crafted to be witty, substantive, and targeted for high engagement.

- **Nodes Involved:**  
  - AI Agent  
  - xAI Grok Chat Model (Language Model node)  
  - Structured Output Parser1  
  - Sticky Note (Documentation)

- **Node Details:**

  - **AI Agent**  
    - Type: LangChain AI Agent  
    - Configuration:  
      - Receives prompt text constructed from `theme` and `cta`.  
      - System message defines “The Carousel Cynic” persona with detailed instructions on JSON output format, style, slide structure, language rules, and length constraints.  
      - Output parser is enabled with a JSON schema example for slides.  
    - Key Expressions:  
      - Prompt: `"=The carousel theme is about: {{ $json.theme }}\n\nThe Call to action (CTA): {{ $json.cta }}"`  
      - System Message: Detailed instructions for slide content generation (7 slides total).  
    - Inputs: From `Fields: Get Theme and CTA`  
    - Outputs: Structured JSON with slides array.  
    - Edge Cases: AI generating invalid JSON, incomplete output, or unexpected format.  
    - Credentials: Uses `xAI Grok` API credentials for language model access.

  - **xAI Grok Chat Model**  
    - Type: Language Model (xAI Grok)  
    - Configuration: Temperature 0.7 and top_p 0.9 for creative but focused output.  
    - Inputs: Queries from AI Agent node.  
    - Outputs: Raw AI-generated text.  
    - Credentials: Requires valid xAI account API key.

  - **Structured Output Parser1**  
    - Type: LangChain Output Parser (Structured)  
    - Configuration: Parses AI raw output using a JSON schema example to enforce structure.  
    - Inputs: Raw AI output.  
    - Outputs: Parsed JSON array of slides with titles and descriptions.

  - **Sticky Note**  
    - Content: Explains this block as AI prompt refinement and content drafting phase.

---

#### 2.3 Content Parsing and Preparation

- **Overview:**  
This block processes the parsed AI content, sets style parameters for text rendering, downloads the background image template from Google Drive, and splits the slide data into batches for sequential image generation.

- **Nodes Involved:**  
  - Params Style Config (Code)  
  - Download file (Google Drive)  
  - Get Slides And Set Params1 (Set)  
  - Split Out (Split Out)  
  - Loop Over Items (Split In Batches)  
  - Sticky Note1 (Documentation)

- **Node Details:**

  - **Params Style Config**  
    - Type: Code (JavaScript)  
    - Configuration:  
      - Assigns the slides from AI output.  
      - Defines style parameters for title and description text such as font size (35-36 px), font color (#ffffff), text positions (X=100, Y~450-514), and max line length (45).  
      - Sets nickname and background image URL (Google Drive file link).  
    - Inputs: From AI Agent output.  
    - Outputs: JSON with slides, style parameters, nickname, and background image URL.

  - **Download file**  
    - Type: Google Drive node (download)  
    - Configuration:  
      - Downloads background template image from Google Drive using `fileId` extracted from the provided URL.  
      - Uses Google Drive OAuth2 credentials.  
    - Inputs: Background image URL from Params Style Config.  
    - Outputs: Binary image data for further image editing.  
    - Edge Cases: Invalid file ID, permission errors, network issues.

  - **Get Slides And Set Params1**  
    - Type: Set  
    - Configuration: Extracts slides array, paramsConfig object, and nickname into workflow variables for clarity and downstream use.  
    - Inputs: Downloaded file binary and Params Style Config output.  
    - Outputs: Prepared JSON with slides and styling parameters.

  - **Split Out**  
    - Type: Split Out  
    - Configuration: Splits the slides array into individual slide objects, while carrying over other fields like paramsConfig, nickname, carouselFolder, isPremiumUser, bucketName. Includes binary data (background image).  
    - Inputs: From Get Slides And Set Params1 output.  
    - Outputs: Individual slide JSONs ready for batch processing.

  - **Loop Over Items**  
    - Type: Split In Batches  
    - Configuration: Processes each slide item sequentially without resetting the batch index.  
    - Inputs: From Split Out.  
    - Outputs: One slide at a time for image generation.

  - **Sticky Note1**  
    - Content: Describes this block as the image generation engine handling content processing, downloading templates, and preparing for image edits.

---

#### 2.4 Image Generation and Upload

- **Overview:**  
This block generates carousel slide images by overlaying slide titles and descriptions onto the downloaded background template using the Edit Image nodes. It then uploads the finalized images to a specified Google Drive folder.

- **Nodes Involved:**  
  - Img 1 - Title (Edit Image)  
  - Img 1 - Description8 (Edit Image)  
  - Upload file (Google Drive upload)  
  - Respond to Webhook  
  - Sticky Note3 (Documentation)

- **Node Details:**

  - **Img 1 - Title**  
    - Type: Edit Image  
    - Configuration:  
      - Writes the slide title text onto the background image.  
      - Font: Arial Bold (from system font path).  
      - Font size, color, position, and max line length dynamically set from `paramsConfig.title` parameters in JSON.  
      - Text source: `{{$json.slides.title}}` (current slide title).  
    - Inputs: Binary image data with slide JSON from Loop Over Items.  
    - Outputs: Image with title text overlaid.

  - **Img 1 - Description8**  
    - Type: Edit Image  
    - Configuration:  
      - Writes the slide description text onto the image output from title node.  
      - Font size, color, position, max line length from `paramsConfig.description`.  
      - Filename parameter set dynamically as slide number (e.g., "1.png").  
      - Text source: `{{$json.slides.description}}`.  
    - Inputs: Image output from Img 1 - Title node.  
    - Outputs: Completed slide image with title and description.

  - **Upload file**  
    - Type: Google Drive node (upload)  
    - Configuration:  
      - Uploads the finalized slide image binary to a Google Drive folder with ID `16uxsnNdNklIxCTf7f85JG6ZQJ2HBXsgd`.  
      - Filename set dynamically from the image binary's fileName property, e.g., "1.png".  
      - Uses Google Drive OAuth2 credentials.  
    - Inputs: Image binary from Img 1 - Description8.  
    - Outputs: Confirmation of upload.  
    - Edge Cases: Upload failures, permission issues.

  - **Respond to Webhook**  
    - Type: Respond to Webhook  
    - Configuration: Sends back the workflow response once all images are processed and uploaded.  
    - Inputs: From Loop Over Items (parallel output path).  
    - Outputs: HTTP response to webhook caller.

  - **Sticky Note3**  
    - Content: Detailed description of the workflow purpose, how it works, credential setup, customization tips, and useful link to Canva background image template for easy style editing:  
      https://www.canva.com/design/DAG5Lh40qks/I-PL6LLfIqZBYXOrZUjYGA/edit?utm_content=DAG5Lh40qks&utm_campaign=designshare&utm_medium=link2&utm_source=sharebutton

---

### 3. Summary Table

| Node Name               | Node Type                      | Functional Role                        | Input Node(s)             | Output Node(s)            | Sticky Note                                                                                              |
|-------------------------|--------------------------------|-------------------------------------|---------------------------|---------------------------|--------------------------------------------------------------------------------------------------------|
| Webhook                 | n8n-nodes-base.webhook         | Input reception via HTTP POST       | -                         | Fields: Get Theme and CTA  | ## 🚀 1. Start Execution This node handles the initial workflow trigger and defines all key parameters |
| When clicking ‘Test workflow’ | n8n-nodes-base.manualTrigger  | Manual trigger for testing          | -                         | Test Params                |                                                                                                        |
| Test Params             | n8n-nodes-base.code             | Provides sample test input           | When clicking ‘Test workflow’ | Fields: Get Theme and CTA  |                                                                                                        |
| Fields: Get Theme and CTA| n8n-nodes-base.set             | Extracts theme, CTA, and nickname    | Webhook, Test Params       | AI Agent                  |                                                                                                        |
| AI Agent                | @n8n/n8n-nodes-langchain.agent | Generates structured slide content   | Fields: Get Theme and CTA  | Params Style Config        | ## 🧠 2. AI Agent: Prompt Refinement and Content Draft The internal AI Agent dynamically refines the user's input prompt and generates content |
| xAI Grok Chat Model     | @n8n/n8n-nodes-langchain.lmChatXAiGrok | Language model for AI Agent          | AI Agent (languageModel)   | AI Agent (outputParser)    |                                                                                                        |
| Structured Output Parser1 | @n8n/n8n-nodes-langchain.outputParserStructured | Parses AI output JSON                 | AI Agent (outputParser)    | AI Agent                  |                                                                                                        |
| Params Style Config     | n8n-nodes-base.code             | Sets slide style parameters and background image info | AI Agent                  | Download file             |                                                                                                        |
| Download file           | n8n-nodes-base.googleDrive      | Downloads background template image | Params Style Config        | Get Slides And Set Params1 |                                                                                                        |
| Get Slides And Set Params1 | n8n-nodes-base.set             | Prepares slides and styling params   | Download file              | Split Out                 |                                                                                                        |
| Split Out               | n8n-nodes-base.splitOut         | Splits slides into individual items | Get Slides And Set Params1 | Loop Over Items           |                                                                                                        |
| Loop Over Items         | n8n-nodes-base.splitInBatches   | Processes each slide sequentially    | Split Out                  | Img 1 - Title, Respond to Webhook |                                                                                                        |
| Img 1 - Title           | n8n-nodes-base.editImage        | Overlays slide title on image        | Loop Over Items            | Img 1 - Description8      |                                                                                                        |
| Img 1 - Description8    | n8n-nodes-base.editImage        | Overlays slide description on image  | Img 1 - Title              | Upload file               |                                                                                                        |
| Upload file             | n8n-nodes-base.googleDrive      | Uploads finished images to Drive     | Img 1 - Description8       | Loop Over Items           |                                                                                                        |
| Respond to Webhook      | n8n-nodes-base.respondToWebhook | Sends HTTP response on completion    | Loop Over Items            | -                         |                                                                                                        |
| Sticky Note             | n8n-nodes-base.stickyNote       | Documentation on AI Agent block      | -                         | -                         | ## 🧠 2. AI Agent: Prompt Refinement and Content Draft                                                   |
| Sticky Note1            | n8n-nodes-base.stickyNote       | Documentation on Image Generation block | -                         | -                         | ## 🎨 3. Image Generation Engine                                                                         |
| Sticky Note2            | n8n-nodes-base.stickyNote       | Documentation on Input Reception block | -                         | -                         | ## 🚀 1. Start Execution                                                                                  |
| Sticky Note3            | n8n-nodes-base.stickyNote       | Documentation on overall workflow and usage | -                         | -                         | # Automated "Viral Style" Carousel Generator for Instagram, Tiktok, Linkedin or X ... [Canva link]       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node:**  
   - Type: `Webhook`  
   - HTTP Method: POST  
   - Path: `ai-carousel-generator`  
   - Response Mode: `Response Node`

2. **Create Manual Trigger Node:**  
   - Type: `Manual Trigger`  
   - Purpose: For testing the workflow manually.

3. **Create Code Node for Test Params:**  
   - Type: `Code` (JavaScript)  
   - Code: Return an object with `theme`, `nickname`, and `cta` fields, e.g.:  
     ```js
     return [{ body: { theme: "Our best AI Carousel generator for n8n creators", nickname: "@myuser", cta: "Send a DM message" } }];
     ```
   - Connect Manual Trigger to this node.

4. **Create Set Node "Fields: Get Theme and CTA":**  
   - Type: `Set`  
   - Assign variables from input JSON body:  
     - `theme` = `={{ $json.body.theme }}`  
     - `cta` = `={{ $json.body.cta }}`  
     - `nickname` = `={{ $json.body.nickname }}`  
   - Connect Webhook and Test Params nodes to this node (merge with OR logic).

5. **Create LangChain AI Agent Node:**  
   - Type: `@n8n/n8n-nodes-langchain.agent`  
   - Prompt: `=The carousel theme is about: {{ $json.theme }}\n\nThe Call to action (CTA): {{ $json.cta }}`  
   - System Message: Use detailed prompt defining “The Carousel Cynic” persona, JSON output format with 7 slides, slide structure, language rules, and no additional commentary.  
   - Enable output parser with JSON schema specifying slides array with title and description.  
   - Connect output to the AI model node.

6. **Create xAI Grok Chat Model Node:**  
   - Type: `@n8n/n8n-nodes-langchain.lmChatXAiGrok`  
   - Parameters: `temperature`=0.7, `topP`=0.9  
   - Provide xAI API credentials.  
   - Connect AI Agent node’s language model input to this node.

7. **Create Structured Output Parser Node:**  
   - Type: `@n8n/n8n-nodes-langchain.outputParserStructured`  
   - JSON Schema Example:  
     ```json
     {
       "slides": [
         {
           "title": "<slide_title>",
           "description": "<slide_description>"
         }
       ]
     }
     ```  
   - Connect AI Agent node’s output parser input to this node.

8. **Create Code Node "Params Style Config":**  
   - Type: `Code`  
   - Code: Return an object containing:  
     - `slides` (from AI output)  
     - `paramsConfig` object with font size, color, text positions for titles and descriptions  
     - `nickname`  
     - `backgroundImage` URL (Google Drive file link)  
   - Connect AI Agent output parser to this node.

9. **Create Google Drive Download Node:**  
   - Operation: Download  
   - File ID: Extracted or set from `backgroundImage` URL in previous node.  
   - Credentials: Google Drive OAuth2  
   - Connect Params Style Config output to this node.

10. **Create Set Node "Get Slides And Set Params1":**  
    - Assign variables:  
      - `slides` = `={{ $json.body.slides }}`  
      - `paramsConfig` = `={{ $json.body.paramsConfig }}`  
      - `nickname` = `={{ $json.body.nickname }}`  
    - Connect Download file node to this node.

11. **Create Split Out Node:**  
    - Field to split out: `slides`  
    - Include fields: `paramsConfig`, `nickname`, `carouselFolder`, `isPremiumUser`, `bucketName`  
    - Include binary data (background image).  
    - Connect Set node to this node.

12. **Create Split In Batches Node "Loop Over Items":**  
    - Options: reset = false  
    - Connect Split Out node to this node.

13. **Create Edit Image Node "Img 1 - Title":**  
    - Operation: Text overlay  
    - Text: `={{ $json.slides.title }}`  
    - Font: Arial Bold (system path `/usr/share/fonts/truetype/msttcorefonts/Arial_Bold.ttf`)  
    - Font size, color, position, max line length from `paramsConfig.title` fields.  
    - Connect Loop Over Items output to this node.

14. **Create Edit Image Node "Img 1 - Description8":**  
    - Operation: Text overlay  
    - Text: `={{ $json.slides.description }}`  
    - Font size, color, position, max line length from `paramsConfig.description` fields.  
    - Filename: `={{ $runIndex + 1 }}.png` (dynamic numbering)  
    - Connect Img 1 - Title node output to this node.

15. **Create Google Drive Upload Node:**  
    - Operation: Upload  
    - Folder ID: Target Google Drive folder for output images  
    - File name: `={{ $binary.data.fileName }}`  
    - Credentials: Google Drive OAuth2  
    - Connect Img 1 - Description8 node output to this node.

16. **Connect Loop Over Items’ second output to Respond to Webhook node:**  
    - Type: Respond to Webhook  
    - Sends final HTTP response when all slides are processed.  

17. **Add Sticky Notes:**  
    - Place sticky notes at relevant sections to document the workflow blocks and usage tips.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                               | Context or Link                                                                                                                                                                                                       |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Automated "Viral Style" Carousel Generator for Instagram, Tiktok, Linkedin or X. This workflow automatically generates high-engagement carousels by combining AI-generated punchy content with image overlay on background templates. Requires credentials for xAI Grok and Google Drive. Customizable fonts, colors, and positions. Includes bonus Canva editable background template. | https://www.canva.com/design/DAG5Lh40qks/I-PL6LLfIqZBYXOrZUjYGA/edit?utm_content=DAG5Lh40qks&utm_campaign=designshare&utm_medium=link2&utm_source=sharebutton                                                      |
| The AI Agent is configured as "The Carousel Cynic," producing 7-slide JSON with strict style and language rules targeting entrepreneurs and creators for social media virality. The system prompt enforces JSON-only output for reliable parsing.                                                                                                                                                              | Internal AI Agent configuration within node "AI Agent"                                                                                                                                                             |
| Fonts used are system fonts (Arial Bold) requiring font availability on the n8n host environment. Image text positions and font properties are configurable via the `Params Style Config` node.                                                                                                                                                                                                              | Image generation block (Edit Image nodes)                                                                                                                                                                          |
| Google Drive nodes require OAuth2 credentials with appropriate permissions to download background templates and upload final images. Folder IDs must be verified and updated according to user’s Drive structure.                                                                                                                                                                                           | Google Drive nodes: "Download file" and "Upload file"                                                                                                                                                             |

---

**Disclaimer:** The provided text is exclusively generated from an automated n8n workflow analysis. It complies strictly with content policies and contains no illegal or protected content. All data processed is legal and public.