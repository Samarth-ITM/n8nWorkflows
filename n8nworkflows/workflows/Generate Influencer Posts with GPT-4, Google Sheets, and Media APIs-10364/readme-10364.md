Generate Influencer Posts with GPT-4, Google Sheets, and Media APIs

https://n8nworkflows.xyz/workflows/generate-influencer-posts-with-gpt-4--google-sheets--and-media-apis-10364


# Generate Influencer Posts with GPT-4, Google Sheets, and Media APIs

### 1. Workflow Overview

This workflow automates the generation of influencer-style social media posts using AI, integrating user-submitted creative assets, large language models, media generation APIs, and Google Sheets for output management. It is designed for marketing teams, content creators, or brands aiming to rapidly produce cohesive image and video posts featuring a digital AI influencer named "Aarna Verse."  

The workflow is logically divided into three main blocks:

- **1.1 Input Reception & Asset Processing**: Captures user inputs via a form, extracts and converts uploaded images to Base64, then uploads these images to a remote storage to obtain accessible URLs.

- **1.2 AI-Driven Post Idea Generation**: Uses a LangChain agent powered by GPT-4 to generate structured JSON prompts for each requested post, reflecting creative direction and brand tone, including image and video prompts.

- **1.3 Media Generation & Output Management**: Splits the AI’s structured output into individual posts, triggers sub-workflows to generate images and videos as needed, and writes the final captions, media URLs, and statuses to a Google Sheet for review.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Asset Processing

**Overview:**  
This initial block collects user inputs through a web form including images and creative directions, processes the uploaded files by extracting their binary data, converts them to Base64, uploads them to a storage service, and aggregates the returned URLs for AI prompt generation.

**Nodes Involved:**  
- On form submission  
- Character Image  
- Setting Image  
- Item Image  
- Merge Inputs  
- Upload Images (Base64→URL)  
- Aggregate Uploaded URLs  

**Node Details:**

- **On form submission**  
  - *Type:* Form Trigger  
  - *Role:* Entry point capturing user inputs from a form titled "Creative Brief" with required image uploads, dropdowns for number of images/videos, radio for aspect ratio, and a text creative direction field.  
  - *Key Parameters:* Form fields include Character Image (mandatory), Setting Image, Item Image, number of images/videos, creative direction text, and aspect ratio.  
  - *Connections:* Outputs to three nodes extracting files from the form’s binary data.  
  - *Edge Cases:* Missing required files or fields, malformed submissions, large file uploads.  

- **Character Image / Setting Image / Item Image**  
  - *Type:* Extract From File  
  - *Role:* Extract binary data from the respective uploaded file fields and store in JSON properties.  
  - *Configuration:* Each configured to operate on the corresponding form field (e.g., "Character_Image") and convert binary to property.  
  - *Connections:* All three connect to "Merge Inputs".  
  - *Edge Cases:* Missing files, corrupted binary data.  

- **Merge Inputs**  
  - *Type:* Merge  
  - *Role:* Combines the three binary-to-JSON extracted inputs into a single data stream for batch processing.  
  - *Configuration:* Set to accept three inputs.  
  - *Connections:* Outputs to "Upload Images (Base64→URL)".  
  - *Edge Cases:* Missing inputs, unbalanced merges.  

- **Upload Images (Base64→URL)**  
  - *Type:* HTTP Request  
  - *Role:* Uploads Base64 encoded images to a remote file storage API, obtaining shareable URLs.  
  - *Configuration:* POST method to a configurable upload endpoint, sending Base64 data and metadata (upload path and filename).  
  - *Expressions:* Base64 data is dynamically extracted from merged input JSON.  
  - *Connections:* Outputs to "Aggregate Uploaded URLs".  
  - *Edge Cases:* Network errors, API authentication failures, upload limits, invalid Base64 data.  

- **Aggregate Uploaded URLs**  
  - *Type:* Aggregate  
  - *Role:* Collects all uploaded image URLs from responses into a single array for use in AI prompt generation.  
  - *Configuration:* Aggregates on the "data.downloadUrl" field from upload responses.  
  - *Connections:* Outputs to "Generate Post Ideas".  
  - *Edge Cases:* Missing URLs, partial upload failures.

---

#### 2.2 AI-Driven Post Idea Generation

**Overview:**  
Using the uploaded asset URLs and user creative input, this block calls a LangChain agent powered by GPT-4 to generate a structured JSON containing detailed post ideas for images and videos. It also uses a structured output parser to enforce JSON schema compliance.

**Nodes Involved:**  
- Generate Post Ideas (LangChain Agent)  
- LLM (OpenAI Chat Model)  
- Structured Output Parser  
- Think (LangChain tool)  
- Split Posts  

**Node Details:**

- **Generate Post Ideas**  
  - *Type:* LangChain Agent Node  
  - *Role:* Uses a complex prompt to generate a structured JSON object describing post titles, captions, image prompts, and video prompts compliant with the Aarna Verse character and creative brief.  
  - *Configuration:*  
    - Dynamic input text interpolates user inputs (number of images/videos, aspect ratio, creative direction) and uploaded image references.  
    - System prompt details character personality, style, voice, and strict output format requirements.  
    - Uses tools: Think, Style Continuity, Context Alignment, Realism Guard for internal reasoning and quality control.  
  - *Connections:* Uses "Think" node as a reasoning sub-tool, outputs to "Structured Output Parser".  
  - *Edge Cases:* Output not matching schema, API rate limits, incomplete or malformed JSON output, logic failures in prompt processing.  

- **LLM (OpenAI Chat Model)**  
  - *Type:* Language Model (OpenAI GPT-4)  
  - *Role:* Provides the underlying language generation capability for the LangChain agent.  
  - *Configuration:* Model set to GPT-4 (gpt-4.1), credentials use an OpenAI API key.  
  - *Connections:* Input from "Generate Post Ideas", output back to it.  
  - *Edge Cases:* API key invalidation, request timeouts, token limits.  

- **Structured Output Parser**  
  - *Type:* LangChain Structured Output Parser  
  - *Role:* Ensures the AI output conforms strictly to the defined JSON schema for posts.  
  - *Configuration:* Manual schema with nested "posts" object, each post containing title, caption, post_type, image_prompt, and video_prompt fields.  
  - *Connections:* Input from "Generate Post Ideas", output to "Split Posts".  
  - *Edge Cases:* Parsing errors, schema mismatches, incomplete data.  

- **Think**  
  - *Type:* LangChain Tool (Think Tool)  
  - *Role:* Internal reasoning helper enabling the agent to reflect on and improve prompt generation quality, especially regarding brand consistency and realism.  
  - *Connections:* Used as a sub-tool inside "Generate Post Ideas".  
  - *Edge Cases:* Logic loops or delays, excessive token usage.  

- **Split Posts**  
  - *Type:* SplitOut  
  - *Role:* Splits the structured JSON posts object into individual post items to process separately.  
  - *Configuration:* Splitting on "output.posts" property.  
  - *Connections:* Outputs to media generation workflows.  
  - *Edge Cases:* Missing or empty posts, malformed split data.

---

#### 2.3 Media Generation & Output Management

**Overview:**  
This block triggers sub-workflows to generate images and videos based on the AI prompts, then records the generated URLs and captions into a Google Sheet. It also conditionally routes video posts to a dedicated video creation workflow.

**Nodes Involved:**  
- Create Image Task (Sub-Workflow)  
- If Video Post (Filter)  
- Create Video Task (Sub-Workflow)  
- Append row in sheet (Google Sheets)  
- Update row in sheet (Google Sheets)  

**Node Details:**

- **Create Image Task**  
  - *Type:* Execute Workflow (Sub-Workflow)  
  - *Role:* Invokes a sub-workflow dedicated to generating images from the provided image prompt and uploaded asset URLs.  
  - *Configuration:*  
    - Inputs include entire original form submission JSON, aggregated uploaded URLs, and the post’s image_prompt string.  
    - Workflow ID references a separate workflow titled "Create Image".  
  - *Connections:* Outputs to "Append row in sheet".  
  - *Edge Cases:* Sub-workflow errors, media generation API failures, timeout, invalid prompt handling.  

- **If Video Post**  
  - *Type:* Filter  
  - *Role:* Checks if the current post is a video and that the status is not "error" before proceeding to video generation.  
  - *Configuration:*  
    - Condition: post_type equals "video" AND status not equal to "error".  
  - *Connections:* True path leads to "Create Video Task".  
  - *Edge Cases:* Missing post_type field, unexpected status values.  

- **Create Video Task**  
  - *Type:* Execute Workflow (Sub-Workflow)  
  - *Role:* Invokes a sub-workflow to generate videos based on the video prompt from the post data.  
  - *Configuration:*  
    - Inputs include the original form submission JSON and the post’s video_prompt string.  
    - Workflow ID references a separate workflow titled "Create Video".  
  - *Connections:* Outputs to "Update row in sheet".  
  - *Edge Cases:* Sub-workflow failures, video generation issues, timeouts, invalid prompts.  

- **Append row in sheet**  
  - *Type:* Google Sheets  
  - *Role:* Appends a new row to a Google Sheet recording the post ID, title, caption, image output URL, and status ("scheduled" or "error").  
  - *Configuration:*  
    - Uses a configurable Google Sheet document ID and worksheet (gid=0).  
    - Status is set based on whether the image result URL is valid.  
    - Columns include id, title, caption, image_output, video_output (blank here), and status.  
  - *Connections:* Outputs to "If Video Post".  
  - *Edge Cases:* Sheet ID misconfiguration, API authentication issues, row write conflicts.  

- **Update row in sheet**  
  - *Type:* Google Sheets  
  - *Role:* Updates the previously appended row with video output URL after video generation completes.  
  - *Configuration:*  
    - Requires configuration of sheet name and document ID matching the append step.  
  - *Connections:* Final node in media generation chain.  
  - *Edge Cases:* Row not found for update, API failures, partial update.

---

### 3. Summary Table

| Node Name             | Node Type                        | Functional Role                                | Input Node(s)                          | Output Node(s)                   | Sticky Note                                                                                                                                                         |
|-----------------------|---------------------------------|------------------------------------------------|--------------------------------------|---------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| On form submission    | Form Trigger                    | Entry point receiving creative brief inputs    | -                                    | Character Image, Setting Image, Item Image | See Sticky Note5 for workflow intro and usage instructions.                                                                                                       |
| Character Image       | Extract From File               | Extract binary data for character image         | On form submission                   | Merge Inputs                    | See Sticky Note4 for character image description with example image.                                                                                              |
| Setting Image         | Extract From File               | Extract binary data for setting image           | On form submission                   | Merge Inputs                    | See Sticky Note6 for setting/background description with example image.                                                                                           |
| Item Image            | Extract From File               | Extract binary data for item/product image      | On form submission                   | Merge Inputs                    | See Sticky Note7 for item/product description with example image.                                                                                                |
| Merge Inputs          | Merge                          | Combine extracted image data for upload batch   | Character Image, Setting Image, Item Image | Upload Images (Base64→URL)     | See Sticky Note for Step 1 overview of content asset upload.                                                                                                     |
| Upload Images (Base64→URL) | HTTP Request                  | Upload Base64 images to remote storage           | Merge Inputs                        | Aggregate Uploaded URLs          | See Sticky Note for Step 1 overview of content asset upload.                                                                                                     |
| Aggregate Uploaded URLs | Aggregate                      | Aggregate URLs returned from image uploads       | Upload Images (Base64→URL)           | Generate Post Ideas             | See Sticky Note1 for Step 2 overview of AI analysis and prompt generation.                                                                                       |
| Generate Post Ideas    | LangChain Agent                | Generate structured JSON post prompts            | Aggregate Uploaded URLs              | Structured Output Parser        | See Sticky Note1 for Step 2 overview.                                                                                                                            |
| LLM (OpenAI Chat Model) | LangChain Language Model       | GPT-4 model powering prompt generation           | Generate Post Ideas                  | Generate Post Ideas             | -                                                                                                                                                                 |
| Structured Output Parser | LangChain Output Parser        | Enforces JSON schema on AI output                 | Generate Post Ideas                  | Split Posts                    | -                                                                                                                                                                 |
| Think                  | LangChain Tool (Think)         | Internal reasoning tool for prompt quality       | Generate Post Ideas (tool call)     | Generate Post Ideas             | -                                                                                                                                                                 |
| Split Posts            | SplitOut                      | Split JSON posts into individual post items      | Structured Output Parser             | Create Image Task               | -                                                                                                                                                                 |
| Create Image Task      | Execute Workflow (Sub-Workflow) | Generate images from prompts                      | Split Posts                        | Append row in sheet             | See Sticky Note2 for Step 3 overview of asset creation and Sheet writing.                                                                                        |
| Append row in sheet    | Google Sheets                  | Append new post data with image URL and status   | Create Image Task                   | If Video Post                  | See Sticky Note2 for Step 3 overview.                                                                                                                            |
| If Video Post          | Filter                        | Route video posts for video generation            | Append row in sheet                 | Create Video Task               | -                                                                                                                                                                 |
| Create Video Task      | Execute Workflow (Sub-Workflow) | Generate videos from prompts                      | If Video Post                      | Update row in sheet             | See Sticky Note2 for Step 3 overview.                                                                                                                            |
| Update row in sheet    | Google Sheets                  | Update post row with video URL after generation   | Create Video Task                  | -                             | See Sticky Note3 for final step and review note.                                                                                                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Form Trigger Node "On form submission"**  
   - Type: Form Trigger  
   - Configure webhook with form titled "Creative Brief"  
   - Add required fields:  
     - File upload: Character Image (required)  
     - File upload: Setting Image (optional)  
     - File upload: Item Image (optional)  
     - Dropdown: How many images? (options 1 to 10, required)  
     - Dropdown: How many Videos? (options 1 to 10, required)  
     - Text: Creative Direction (required)  
     - Radio: Aspect Ratio (options "9:16", "16:9", required)  

2. **Add three "Extract From File" nodes**  
   - For each of Character Image, Setting Image, Item Image:  
     - Operation: binaryToProperty  
     - Binary Property Name: match form field names (e.g., "Character_Image")  
   - Connect outputs of "On form submission" to these extraction nodes.  

3. **Add "Merge" node "Merge Inputs"**  
   - Number of inputs: 3  
   - Connect outputs of the three extract nodes into "Merge Inputs".  

4. **Add HTTP Request node "Upload Images (Base64→URL)"**  
   - Method: POST  
   - URL: Set to your file upload API endpoint (replace placeholder <<FILE_UPLOAD_BASE>>)  
   - Body parameters:  
     - base64Data: `{{$json.data}}` (Base64 from merged input)  
     - uploadPath: "documents/uploads"  
     - fileName: `{{$now}}.jpg` (timestamp-based filename)  
   - Connect "Merge Inputs" output here.  

5. **Add "Aggregate" node "Aggregate Uploaded URLs"**  
   - Aggregate on field: `data.downloadUrl`  
   - Connect output of HTTP Request here.  

6. **Add LangChain Agent node "Generate Post Ideas"**  
   - Paste the complex prompt text as given, referencing form data and uploaded URLs via expressions:  
     - Number of image posts: `{{ $('On form submission').item.json['How many images?'] }}`  
     - Number of video posts: `{{ $('On form submission').item.json['How many Videos?'] }}`  
     - Aspect ratio: `{{ $('On form submission').item.json['Aspect Ratio'] }}`  
     - Creative direction: `{{ $('On form submission').item.json['Creative Direction'] }}`  
     - Image references: filenames from uploaded images  
   - Configure system prompt to enforce Aarna Verse character style and output schema.  
   - Connect "Aggregate Uploaded URLs" output here.  

7. **Add LLM (OpenAI Chat Model) node**  
   - Model: GPT-4 (gpt-4.1)  
   - Configure with OpenAI API credentials  
   - Connect as the language model for "Generate Post Ideas".  

8. **Add Structured Output Parser**  
   - Set manual schema matching expected JSON posts format (title, caption, post_type, image_prompt, video_prompt).  
   - Connect output from "Generate Post Ideas" to this node.  

9. **Add "Think" LangChain Tool node**  
   - Used internally by "Generate Post Ideas" for reasoning (no direct connection needed).  

10. **Add "SplitOut" node "Split Posts"**  
    - Field to split: `output.posts`  
    - Connect output of Structured Output Parser here.  

11. **Add Execute Workflow node "Create Image Task"**  
    - Workflow ID: reference the image generation sub-workflow  
    - Pass inputs: form submission JSON, aggregated uploaded URLs, current post’s image_prompt  
    - Connect from "Split Posts".  

12. **Add Google Sheets node "Append row in sheet"**  
    - Operation: Append  
    - Document ID: Paste your Google Sheet ID  
    - Sheet Name: e.g., "gid=0" or as appropriate  
    - Columns mapped:  
      - id: `{{ $execution.id }}_{{ $json.title }}`  
      - title: `{{ $json.title }}`  
      - caption: `{{ $json.caption }}`  
      - image_output: `{{ $json.image_result }}` (from sub-workflow output)  
      - status: conditional on whether image_output URL exists ("scheduled" or "error")  
    - Connect output of "Create Image Task" here.  

13. **Add Filter node "If Video Post"**  
    - Condition: post_type equals "video" AND status not "error"  
    - Connect output of "Append row in sheet" here.  

14. **Add Execute Workflow node "Create Video Task"**  
    - Workflow ID: reference the video generation sub-workflow  
    - Pass inputs: form submission JSON, current post’s video_prompt  
    - Connect true output of filter here.  

15. **Add Google Sheets node "Update row in sheet"**  
    - Operation: Update  
    - Document ID and Sheet Name must match append node  
    - Update the corresponding row with video_output URL once video is generated  
    - Connect output of "Create Video Task" here.  

---

### 5. General Notes & Resources

| Note Content                                                                                                                     | Context or Link                                                                                           |
|---------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------|
| This workflow turns brand product images and creative assets into AI-generated influencer-style posts with captions and media. | See Sticky Note5 for comprehensive workflow introduction and usage instructions.                         |
| Images represent key assets: Character Image, Setting Image, and Item/Product Image, each with example images linked inline.    | See Sticky Notes4, 6, and 7 for visual references and descriptions of each asset type.                   |
| The LangChain Agent uses a complex system prompt ensuring AI outputs adhere to brand identity, style, and realistic aesthetics. | Prompt is embedded in "Generate Post Ideas" node; includes tools for style continuity, context, and realism. |
| Generated posts are saved in Google Sheets for easy review and further processing.                                               | Final results appear in configured Google Sheet; see Sticky Note3.                                       |
| Requires credentials: OpenAI API key, Google Sheets OAuth2, and access to a file upload API endpoint for storing images.       | Credentials must be set in respective nodes before execution.                                            |
| AI-generated content is for creative inspiration and should be reviewed before publishing.                                       | Workflow includes disclaimers and best practice notes for content review.                                |

---

**Disclaimer:** The provided content originates exclusively from an automated workflow created with n8n, adhering strictly to current content policies without any illegal, offensive, or protected material. All processed data is legal and publicly accessible.