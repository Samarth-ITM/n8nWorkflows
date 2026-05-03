Create AI-Powered Social Media Content for LinkedIn & Facebook with Image Generation & Human Approval

https://n8nworkflows.xyz/workflows/create-ai-powered-social-media-content-for-linkedin---facebook-with-image-generation---human-approval-8398


# Create AI-Powered Social Media Content for LinkedIn & Facebook with Image Generation & Human Approval

### 1. Workflow Overview

This workflow automates the creation, refinement, approval, and scheduling of AI-generated social media content tailored for LinkedIn and Facebook platforms, with integrated image generation and human approval loops. It is designed for marketing teams or agencies aiming to efficiently generate data-driven, brand-aligned social media campaigns with iterative visual content feedback.

The workflow consists of the following logical blocks:

- **1.1 Input Reception:** Captures brand and campaign variables via a web form.
- **1.2 AI Content Generation:** Uses a LangChain AI agent to generate a week’s worth of social media posts (3 LinkedIn + 3 Facebook), including detailed image prompts and scheduling data.
- **1.3 Output Structuring & Splitting:** Parses and splits AI-generated JSON output into individual posts for processing.
- **1.4 Platform Routing:** Routes posts based on platform (LinkedIn or Facebook) to separate image generation and approval paths.
- **1.5 Image Generation:** Calls Google Imagen4 API to generate images per post style and prompt.
- **1.6 Image Download & Processing:** Downloads generated images, and supports optional image editing using the Nano Banana model based on user feedback.
- **1.7 Human Approval Loop:** Sends images and posts to Slack for human review, collects approval or feedback, and if rejected, triggers image regeneration with feedback applied.
- **1.8 File Upload & Logging:** Uploads approved images to Google Drive and logs post data in a Google Sheet.
- **1.9 Scheduling:** Schedules approved posts on social media platforms via the Late API.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:** Receives campaign variables from a user-filled web form to customize content generation.
- **Nodes Involved:**  
  - *On form submission*  
  - *Edit Fields*  
  - *Sticky Note* (variable definitions)

- **Node Details:**

  - **On form submission**  
    - Type: Form Trigger  
    - Role: Entry point for user input  
    - Configuration: Form fields for brand name, industry, demographics, location, value proposition, customer challenge, desired outcome, brand voice, specific topics, and company website.  
    - Outputs data object with all variables for downstream processing.  
    - Edge cases: Missing required fields, malformed inputs, webhook misconfiguration.

  - **Edit Fields**  
    - Type: Set (data assignment)  
    - Role: Assigns and normalizes form inputs into workflow variables for AI prompt use.  
    - Configuration: Copies all form fields to named variables used in AI agent.  
    - Inputs: Data from form submission.  
    - Outputs: Structured variables for AI node.  
    - Edge cases: Expression evaluation errors if variables missing.

  - **Sticky Note**  
    - Content: Lists required variables for user clarity.

---

#### 2.2 AI Content Generation

- **Overview:** Generates a full weekly social media campaign (6 posts, with copy, image prompts, and scheduling info) using a sophisticated AI agent with embedded strategic marketing logic and research capabilities.
- **Nodes Involved:**  
  - *AI Agent2* (LangChain Agent)  
  - *Structured Output Parser1* (structured JSON parsing)  
  - *Simple Memory1* (session memory for context)

- **Node Details:**

  - **AI Agent2**  
    - Type: LangChain Agent  
    - Role: Core AI text generation node producing detailed campaign JSON with 6 posts including LinkedIn and Facebook variants.  
    - Configuration:  
      - Prompt includes dynamic variables (brand, voice, date).  
      - System message embeds detailed instructions for strategic content creation, frameworks (PAS, StoryBrand, BAB), platform tone requirements, visual style definitions, output JSON schema.  
      - Uses web search tool optionally for real-time data enrichment.  
    - Inputs: Brand variables from Edit Fields, session memory.  
    - Outputs: JSON with campaign theme, posts array (each with platform, text, style, image prompt, overlay text, posting date).  
    - Edge cases: API or model errors, prompt parsing failures.

  - **Structured Output Parser1**  
    - Type: LangChain Output Parser  
    - Role: Validates and parses AI output into structured JSON for further processing.  
    - Configuration: Auto-fix enabled for minor inconsistencies, validates against example schema.  
    - Inputs: AI Agent output.  
    - Outputs: Clean JSON object accessible downstream.  
    - Edge cases: Parsing errors on malformed output.

  - **Simple Memory1**  
    - Type: LangChain Memory Buffer  
    - Role: Maintains session context for AI agent continuity.  
    - Inputs/Outputs: Linked to AI Agent2.  
    - Edge cases: Memory overflow, session ID mismanagement.

---

#### 2.3 Output Structuring & Splitting

- **Overview:** Splits the AI-generated posts array into individual items for parallel processing.
- **Nodes Involved:**  
  - *Split Out3*  
  - *Sticky Note3*

- **Node Details:**

  - **Split Out3**  
    - Type: Split Out  
    - Role: Decomposes the posts array JSON into individual post objects with campaign theme attached.  
    - Inputs: Parsed AI output JSON.  
    - Outputs: Individual post JSON records.  
    - Edge cases: Empty posts array, data inconsistency.

  - **Sticky Note3**  
    - Content: Describes splitting of array into single items.

---

#### 2.4 Platform Routing

- **Overview:** Routes each post to a different processing branch based on its platform (LinkedIn or Facebook).
- **Nodes Involved:**  
  - *Switch*  
  - *Sticky Note9*

- **Node Details:**

  - **Switch**  
    - Type: Switch node  
    - Role: Directs posts to LinkedIn or Facebook branches.  
    - Configuration: Checks 'platform' property of each post.  
    - Inputs: Single post JSON from Split Out3.  
    - Outputs: Two branches - linked to LinkedIn or Facebook processing.  
    - Edge cases: Unknown platform values.

  - **Sticky Note9**  
    - Content: Indicates platform-based switch.

---

#### 2.5 Image Generation

- **Overview:** Generates images for each post using Google Imagen4 API, applying style and overlay text from AI output.
- **Nodes Involved:**  
  - *Image Generation* (LinkedIn branch)  
  - *Image Generation 2* (Facebook branch)  
  - *Sticky Note2*, *Sticky Note13* (Google Imagen4 related)

- **Node Details:**

  - **Image Generation / Image Generation 2**  
    - Type: HTTP Request (POST)  
    - Role: Calls Google Imagen4 Ultra API with prompt describing style, overlay text, and aspect ratio for image creation.  
    - Configuration:  
      - JSON body dynamically constructed from post data.  
      - 'Prefer: wait' header ensures synchronous response.  
    - Inputs: Post image_prompt, style, overlay_text, aspect_ratio.  
    - Outputs: URL or identifier of generated image.  
    - Edge cases: API rate limits, invalid prompt formats, network errors.

  - **Sticky Notes**  
    - Describe that these nodes are responsible for image generation using Google Imagen4.

---

#### 2.6 Image Download & Processing

- **Overview:** Downloads generated images, optionally edits them based on user feedback through Nano Banana model, and manages final image URL selection.
- **Nodes Involved:**  
  - *Download Image 1st / Download Image 1st 2*  
  - *Merge Image Paths / Merge Image Paths1* (if original or edited)  
  - *Edit Image Nano Banana / Edit Image Nano Banana2* (image editing via Replicate API)  
  - *Set Original Image URL / Set Original Image URL1*  
  - *Set Edited Image URL / Set Edited Image URL1*  
  - *Sticky Note6, Sticky Note7, Sticky Note8, Sticky Note20, Sticky Note22, Sticky Note23, Sticky Note24*

- **Node Details:**

  - **Download Image 1st / Download Image 1st 2**  
    - Type: HTTP Request (GET)  
    - Role: Downloads the original generated image file for further processing or feedback.  
    - Inputs: URL of generated image from Google Imagen4.  
    - Outputs: Image binary or URL for next steps.  
    - Edge cases: Download failures, invalid URLs.

  - **Merge Image Paths / Merge Image Paths1**  
    - Type: If node  
    - Role: Decides whether to use original or edited image URL based on metadata flag 'image_source'.  
    - Inputs: Image source flag and URLs.  
    - Outputs: Final image URL for download or feedback.  
    - Edge cases: Missing or malformed source flag.

  - **Edit Image Nano Banana / Edit Image Nano Banana2**  
    - Type: HTTP Request (POST)  
    - Role: Calls Replicate’s Nano Banana model to edit images based on user feedback text and image input.  
    - Inputs: User feedback text and current image URL.  
    - Outputs: URL of edited image.  
    - Edge cases: API limits, invalid feedback, model errors.

  - **Set Original Image URL / Set Original Image URL1**  
    - Type: Set node  
    - Role: Marks image as original and sets final_image_url variable accordingly.  
    - Inputs: Image generation output.  
    - Outputs: Metadata for routing.  

  - **Set Edited Image URL / Set Edited Image URL1**  
    - Type: Set node  
    - Role: Marks image as edited and sets final_image_url variable accordingly.  

  - **Sticky Notes**  
    - Explain the purpose and flow of these nodes, especially regarding image editing and selection logic.

---

#### 2.7 Human Approval Loop

- **Overview:** Sends posts with images to Slack for human review; based on approval or feedback, either proceeds or triggers image regeneration.
- **Nodes Involved:**  
  - *Send message and wait for response* (Facebook branch)  
  - *Send message and wait for response2* (LinkedIn branch)  
  - *Switch8* and *Switch7* (handle approval conditions)  
  - *Sticky Note5*, *Sticky Note15*

- **Node Details:**

  - **Send message and wait for response / Send message and wait for response2**  
    - Type: Slack node (sendAndWait)  
    - Role: Sends the post content and image to a Slack channel requesting approval or review with optional feedback and file upload.  
    - Configuration:  
      - Custom form with radio buttons for 'Approved' or 'Review'.  
      - Feedback text and optional file attachment accepted.  
      - OAuth2 authentication for Slack.  
    - Inputs: Post content and image URL.  
    - Outputs: Approval status and feedback.  
    - Edge cases: Slack API errors, user non-response, invalid form data.

  - **Switch8 / Switch7**  
    - Type: Switch nodes  
    - Role: Branch logic based on approval response:  
      - If "Approved": proceed to upload and scheduling.  
      - If "Review": trigger editing via Nano Banana and resend for approval.  
    - Edge cases: Unexpected response values.

  - **Sticky Notes**  
    - Describe the human-in-the-loop approval process and iterative feedback cycle.

---

#### 2.8 File Upload & Logging

- **Overview:** Uploads approved images to Google Drive and logs post details into a Google Sheet for record keeping.
- **Nodes Involved:**  
  - *Upload file / Upload file1*  
  - *Append or update row in sheet8 / Append or update row in sheet*  
  - *Sticky Note17, Sticky Note18, Sticky Note19, Sticky Note10*

- **Node Details:**

  - **Upload file / Upload file1**  
    - Type: Google Drive node  
    - Role: Uploads the approved image file to a designated folder in Google Drive.  
    - Configuration: Uses dynamic naming based on image id and timestamp.  
    - Inputs: Downloaded image file.  
    - Outputs: Google Drive file metadata including ID and URL.  
    - Edge cases: Google Drive API limits, permission errors.

  - **Append or update row in sheet8 / Append or update row in sheet**  
    - Type: Google Sheets node  
    - Role: Logs post text, platform, posting date, image URL, image prompt, and image ID into a Google Sheet.  
    - Configuration: Matches on image ID for update or appends if new.  
    - Inputs: Post data and Google Drive file metadata.  
    - Edge cases: Sheet access errors, data format mismatches.

  - **Sticky Notes**  
    - Clarify data storage and upload steps.

---

#### 2.9 Scheduling

- **Overview:** Schedules the approved posts on social media platforms via the Late API.
- **Nodes Involved:**  
  - *Schedule Post / Schedule Post1*  
  - *Sticky Note11, Sticky Note12*

- **Node Details:**

  - **Schedule Post / Schedule Post1**  
    - Type: HTTP Request (POST)  
    - Role: Sends scheduled post requests to Late API with post content, scheduled date/time, timezone, and platform account ID.  
    - Configuration: Requires user to replace placeholder `accountId` with actual social media account IDs.  
    - Inputs: Post text and date from Google Sheets logging.  
    - Outputs: API response confirming scheduling.  
    - Edge cases: API authentication errors, invalid scheduling data, rate limits.

  - **Sticky Notes**  
    - Provide instructions for configuring Late API account IDs.

---

### 3. Summary Table

| Node Name                     | Node Type                            | Functional Role                         | Input Node(s)           | Output Node(s)                  | Sticky Note                                                                                   |
|-------------------------------|------------------------------------|---------------------------------------|------------------------|-------------------------------|-----------------------------------------------------------------------------------------------|
| On form submission             | Form Trigger                       | Receive brand & campaign variables    | -                      | Edit Fields                   | Please fill variables to generate weekly content                                              |
| Edit Fields                   | Set                               | Normalize input variables for AI      | On form submission      | AI Agent2                    |                                                                                               |
| AI Agent2                    | LangChain Agent                    | Generate AI social media campaign     | Edit Fields, Simple Memory1 | Structured Output Parser1      | Nova AI Agent creates 6 posts with detailed instructions                                     |
| Structured Output Parser1      | LangChain Output Parser            | Parse AI output JSON                   | AI Agent2               | Split Out3                   |                                                                                               |
| Simple Memory1                | LangChain Memory Buffer            | Maintain session context               | -                      | AI Agent2                    |                                                                                               |
| Split Out3                   | Split Out                         | Split campaign posts into items       | Structured Output Parser1 | Switch                      | Split array into single posts                                                                 |
| Switch                       | Switch                           | Route posts by platform                | Split Out3              | Image Generation / Image Generation 2 | Switch between LinkedIn and Facebook/Instagram                                           |
| Image Generation             | HTTP Request (Google Imagen4)      | Generate LinkedIn images               | Switch (LinkedIn branch) | Download Image 1st           | Google Imagen4 for image generation                                                          |
| Image Generation 2           | HTTP Request (Google Imagen4)      | Generate Facebook images               | Switch (Facebook branch) | Download Image 1st 2         | Google Imagen4 for image generation                                                          |
| Download Image 1st           | HTTP Request                      | Download LinkedIn generated image     | Image Generation        | Loop Over Items              | Download file                                                                                |
| Download Image 1st 2         | HTTP Request                      | Download Facebook generated image     | Image Generation 2      | Loop Over Items1             | Download file                                                                                |
| Loop Over Items              | Split In Batches                  | Batch processing of LinkedIn images   | Download Image 1st      | Send message and wait for response2 |                                                                                         |
| Loop Over Items1             | Split In Batches                  | Batch processing of Facebook images   | Download Image 1st 2    | Send message and wait for response |                                                                                         |
| Send message and wait for response2 | Slack (sendAndWait)            | Send LinkedIn post for approval       | Loop Over Items         | Switch7                     | Send images for approval, iterative feedback loop                                            |
| Send message and wait for response | Slack (sendAndWait)            | Send Facebook post for approval       | Loop Over Items1        | Switch8                     | Send images for approval, iterative feedback loop                                            |
| Switch7                     | Switch                           | Handle LinkedIn approval/review       | Send message and wait for response2 | Set Original Image URL / Edit Image Nano Banana2 | Switch to Approved or Review                                                  |
| Switch8                     | Switch                           | Handle Facebook approval/review       | Send message and wait for response | Set Original Image URL1 / Edit Image Nano Banana | Switch to Approved or Review                                                  |
| Edit Image Nano Banana2      | HTTP Request (Replicate Nano Banana) | Edit LinkedIn image based on feedback | Switch7 (Review path)   | Set Edited Image URL          | NanoBanana for image editing after user's feedback                                           |
| Edit Image Nano Banana       | HTTP Request (Replicate Nano Banana) | Edit Facebook image based on feedback | Switch8 (Review path)   | Set Edited Image URL1         | NanoBanana for image editing after user's feedback                                           |
| Set Original Image URL       | Set                               | Mark LinkedIn image as original       | Switch7 (Approved path) | Merge Image Paths             | Switch to Edited or Original                                                                |
| Set Edited Image URL         | Set                               | Mark LinkedIn image as edited          | Edit Image Nano Banana2  | Merge Image Paths             | Switch to Edited or Original                                                                |
| Merge Image Paths            | If                                | Choose between original or edited image | Set Original Image URL / Set Edited Image URL | Download Image File / Send message and wait for response2 |                                                                                 |
| Download Image File          | HTTP Request                      | Download final LinkedIn image file    | Merge Image Paths        | Upload file                  | Download final file                                                                         |
| Upload file                 | Google Drive                      | Upload LinkedIn image to Drive        | Download Image File      | Append or update row in sheet8 | Upload file to Google Drive                                                                |
| Append or update row in sheet8 | Google Sheets                    | Log LinkedIn post data                 | Upload file             | Schedule Post                | Save data in spreadsheet                                                                   |
| Schedule Post               | HTTP Request (Late API)            | Schedule LinkedIn post                 | Append or update row in sheet8 | Loop Over Items             | Schedule Post with Late API on LinkedIn                                                    |
| Set Original Image URL1      | Set                               | Mark Facebook image as original       | Switch8 (Approved path) | Merge Image Paths1            | Switch to Edited or Original                                                                |
| Set Edited Image URL1        | Set                               | Mark Facebook image as edited          | Edit Image Nano Banana   | Merge Image Paths1            | Switch to Edited or Original                                                                |
| Merge Image Paths1           | If                                | Choose between original or edited image | Set Original Image URL1 / Set Edited Image URL1 | Download Image File1 / Send message and wait for response |                                                                                 |
| Download Image File1         | HTTP Request                      | Download final Facebook image file    | Merge Image Paths1       | Upload file1                 | Download final file                                                                         |
| Upload file1                | Google Drive                      | Upload Facebook image to Drive        | Download Image File1     | Append or update row in sheet | Upload file to Google Drive                                                                |
| Append or update row in sheet | Google Sheets                    | Log Facebook post data                 | Upload file1            | Schedule Post1               | Save data in spreadsheet                                                                   |
| Schedule Post1              | HTTP Request (Late API)            | Schedule Facebook post                 | Append or update row in sheet | Loop Over Items1            | Schedule Post with Late API on Facebook/Instagram                                          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Form Trigger node ("On form submission")**  
   - Configure a webhook form with fields: BRAND_NAME, INDUSTRY, TARGET_DEMOGRAPHICS, TARGET_LOCATION, PRIMARY_VALUE_PROPOSITION, CUSTOMER_CHALLENGE, DESIRED_OUTCOME, BRAND_VOICE, SPECIFIC_TOPICS_TO_FOCUS (optional), COMPANY_WEBSITE (optional).  
   - Set required fields as appropriate.

2. **Add a Set node ("Edit Fields")**  
   - Map each form field to a workflow variable with the same name.  
   - This normalizes inputs for AI prompt usage.

3. **Add a LangChain Agent node ("AI Agent2")**  
   - Use LangChain Agent type.  
   - Configure the prompt with embedded variables (e.g., {{ $json.BRAND_NAME }}) and instructions to generate 6 social media posts (3 LinkedIn, 3 Facebook) with detailed image prompts, overlay text, and posting dates.  
   - Include system message with strategic instructions and frameworks (PAS, StoryBrand, BAB).  
   - Attach a Simple Memory node for session context.  
   - Connect "Edit Fields" output to this node’s input.

4. **Add a LangChain Output Parser ("Structured Output Parser1")**  
   - Set to parse the AI agent’s JSON output.  
   - Enable auto-fix to handle minor format issues.  
   - Connect AI Agent2 output to this parser.

5. **Add a Split Out node ("Split Out3")**  
   - Configure to split the 'posts' array from the parsed output into individual items.  
   - This allows parallel processing per post.

6. **Add a Switch node ("Switch")**  
   - Route each post by its 'platform' field to separate branches: 'linkedin' and 'facebook'.

7. **For each platform branch:**

   - **Image Generation node (HTTP Request):**  
     - POST to Google Imagen4 Ultra API endpoint.  
     - JSON body includes: prompt (post image prompt + style + overlay text), aspect_ratio.  
     - Set header 'Prefer: wait' for synchronous response.  
     - Credentials: Google Imagen API or equivalent HTTP header auth.

   - **Download Image node (HTTP Request):**  
     - GET request to the image URL returned by image generation.  
     - Fetch the generated image for further use.

   - **Split In Batches node:**  
     - Allows batch processing of images for approval.

   - **Slack node ("Send message and wait for response"):**  
     - Send the post text and image to a Slack channel.  
     - Use a custom form with radio options 'Approved' or 'Review', optional feedback text, and file upload.  
     - OAuth2 credentials setup required.

   - **Switch node:**  
     - If approval is 'Approved', proceed to upload and scheduling.  
     - If 'Review', trigger image editing.

   - **If 'Review':**  
     - HTTP Request node to call Replicate Nano Banana model for image editing.  
     - Pass user feedback and current image URL.  
     - Receive edited image URL.

   - **Set nodes:**  
     - Mark images as 'original' or 'edited' by setting metadata for routing.

   - **If node:**  
     - Select between original or edited image URL for final download.

   - **Download final image node:**  
     - Download the selected final image (original or edited).

   - **Google Drive upload node:**  
     - Upload the final image to a designated Google Drive folder.  
     - Use dynamic file naming based on image ID and timestamp.

   - **Google Sheets node:**  
     - Append or update rows with post data: date, text, image URL, platform, image prompt, image ID.

   - **HTTP Request node (Late API):**  
     - Schedule the post for publishing on the corresponding social platform.  
     - Configure with content, scheduled date/time, timezone, and platform account ID (replace placeholders with actual IDs).

8. **Repeat image generation and approval loops until all posts are approved and scheduled.**

9. **Add Sticky Notes throughout** for documentation, variable definition, and instruction clarity.

---

### 5. General Notes & Resources

| Note Content                                                                                         | Context or Link                                                                                      |
|----------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| Defines essential brand variables like BRAND_NAME, INDUSTRY, TARGET_DEMOGRAPHICS, etc.               | Sticky Note near form trigger node                                                                |
| Nova AI Agent creates 6 posts with platform-specific strategy, tone, and output JSON schema.        | Sticky Note near AI Agent                                                                           |
| Google Imagen4 is used for AI-powered image generation with synchronous API calls.                   | Sticky Notes near Image Generation nodes                                                           |
| Nano Banana model from Replicate is used for image editing based on human feedback.                  | Sticky Notes near image editing nodes                                                              |
| Slack channel used for human approval with custom form for approval status and feedback collection. | Slack nodes with OAuth2 credentials                                                                |
| Late API used for scheduling posts, requires user-specific social media account IDs for operation.  | Sticky Notes near scheduling nodes                                                                 |
| Google Drive and Google Sheets used for file storage and logging post metadata respectively.         | Google Drive and Sheets nodes with OAuth2 credentials                                              |
| Workflow built by Dídac Fernandez Girona, version 3.2, dated August 20, 2025, for strategic marketing. | System message in AI Agent2 node                                                                    |
| Example JSON output schema and framework instructions embedded in AI agent prompt for consistent generation. | Part of AI Agent system message                                                                     |

---

**Disclaimer:** The provided description and analysis are based exclusively on the automated n8n workflow JSON shared. All data and integrations comply with legal and ethical standards.