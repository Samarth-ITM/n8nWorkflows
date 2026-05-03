Human-in-the-Loop Post Designer with Mistral AI, ImageKit, and LinkedIn Publishing

https://n8nworkflows.xyz/workflows/human-in-the-loop-post-designer-with-mistral-ai--imagekit--and-linkedin-publishing-6204


# Human-in-the-Loop Post Designer with Mistral AI, ImageKit, and LinkedIn Publishing

### 1. Workflow Overview

This workflow implements a **Human-in-the-Loop Post Designer** for generating, refining, and publishing LinkedIn posts using AI-generated content, image processing, and manual approval. It targets content creators or marketers who want to automate LinkedIn post creation while maintaining human control over final approval and revisions.

The workflow consists of the following logical blocks:

- **1.1 Input Reception:** Captures user post input via a chat message trigger.
- **1.2 AI Content Generation and Formatting:** Cleans the input, generates banner text, and prepares an image banner with overlay text.
- **1.3 Image Processing and Storage:** Uses ImageKit to create a banner image with dynamic text, then uploads it to an S3 bucket.
- **1.4 Human Approval Loop:** Sends the generated content for human approval via Gmail, collects feedback, and allows revision requests.
- **1.5 Content Conversion and Publishing:** Converts the approved post to HTML and publishes it on LinkedIn.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:** Listens for incoming chat messages containing the initial LinkedIn post content.
- **Nodes Involved:** 
  - When chat message received
  - Edit Fields
  - AI Agent

- **Node Details:**

  - **When chat message received**
    - Type: `chatTrigger` (LangChain)
    - Role: Webhook that triggers workflow on receiving a chat message, allows file uploads.
    - Configuration: Accepts chat input as JSON.
    - Outputs: JSON including chatInput.
    - Failures: Webhook timeout or malformed input.
  
  - **Edit Fields**
    - Type: `Set`
    - Role: Assigns the chatInput string to a variable `post` for downstream processing.
    - Configuration: Sets `post = {{$json.chatInput}}`.
    - Input: From chat trigger.
    - Output: JSON with `post`.
  
  - **AI Agent**
    - Type: `LangChain Agent`
    - Role: Cleans the input post text (removes formatting issues, preserves emojis) and generates a catchy banner text split into lines.
    - Configuration Highlights:
      - System prompt defines role and instructions for cleanup and banner text generation.
      - Output is a JSON object with `postclean`, `banner_text.line1`, and `banner_text.line2`.
      - Spaces are URL-encoded as `%20`.
    - Input: `post` field from previous node.
    - Output: Parsed JSON with cleaned post and banner text.
    - Failures: Parsing errors, prompt timeout, API auth issues.

#### 1.2 AI Content Generation and Formatting

- **Overview:** Parses AI output, prepares an overlay image URL with banner text, and sets up variables for downstream image generation.
- **Nodes Involved:** 
  - Structured Output Parser
  - Edit Fields6
  - HTTP Request6

- **Node Details:**

  - **Structured Output Parser**
    - Type: `outputParserStructured` (LangChain)
    - Role: Parses AI Agent output into structured JSON with `postclean`, `line1`, `line2`.
    - Configuration: JSON schema example provided for validation.
    - Input: AI Agent output.
    - Output: Structured JSON for use in subsequent nodes.
  
  - **Edit Fields6**
    - Type: `Set`
    - Role: Extracts and sets variables for image generation:
      - `line1` and `line2` for banner text lines.
      - Generates a `safeName` string combining the first word of `line1` and current date (formatted as YYYYMMDD) for naming image files.
    - Input: Parsed AI output.
  
  - **HTTP Request6**
    - Type: `HTTP Request`
    - Role: Calls ImageKit API to generate an image with overlay text (banner lines).
    - Configuration:
      - URL dynamically constructed with ImageKit transformations applying the `line1` and `line2` texts on the image.
      - Uses GET request, expects binary/image response.
    - Input: from Edit Fields6.
    - Failures: HTTP errors, invalid URL or transformation strings, ImageKit API limits.

#### 1.3 Image Processing and Storage

- **Overview:** Uploads the generated image from ImageKit to an S3 bucket for persistent storage and dynamic access.
- **Nodes Involved:** 
  - S32
  - Merge

- **Node Details:**

  - **S32**
    - Type: `S3`
    - Role: Uploads the image file using the generated `safeName` as file name into the `rag` bucket.
    - Configuration: 
      - Uses S3 credentials.
      - Operation: Upload.
      - Filename: from `safeName`.
    - Input: Image binary output from HTTP Request6.
    - Failures: S3 connectivity issues, permission errors, file size limits.
  
  - **Merge**
    - Type: `Merge`
    - Role: Combines the S3 upload output with the original post content to pass full data down the line.
    - Configuration: Combine mode: combine all inputs.
    - Input: From S32 and Edit Fields8 (later in workflow).
  
#### 1.4 Human Approval Loop

- **Overview:** Sends the generated post content for manual approval via Gmail, processes approval or revision feedback, and optionally loops for revisions.
- **Nodes Involved:** 
  - Send Content for Approval1
  - Approval Result1
  - Revision based on feedback
  - Edit Fields (reused)

- **Node Details:**

  - **Send Content for Approval1**
    - Type: `Gmail`
    - Role: Sends an email to a user requesting approval of the generated post.
    - Configuration:
      - Recipient email dynamically set.
      - Subject: "Approval Required for LinkedIn post".
      - Message contains the post output.
      - Uses a custom form with dropdown options: Yes, No, Cancel, plus feedback textarea.
      - Waits for response.
    - Credentials: Gmail OAuth2.
    - Failures: Email sending errors, OAuth token expiry, user non-response.
  
  - **Approval Result1**
    - Type: `Switch`
    - Role: Routes workflow based on approval response:
      - Yes → proceeds to post publishing.
      - No → stops or alternative flow (not explicitly connected).
      - Cancel → triggers revision.
    - Conditions based on the form response.
  
  - **Revision based on feedback**
    - Type: `LangChain Agent`
    - Role: Performs post revision based on user feedback.
    - Configuration:
      - Input includes original cleaned post and user feedback.
      - Instructions to make minimal changes preserving tone and structure.
    - Output: Revised post text.
    - Failures: AI model errors, prompt failures.
  
  - **Edit Fields** (re-used)
    - Role: Resets or updates the `post` variable with the revised text to restart AI processing if needed.

#### 1.5 Content Conversion and Publishing

- **Overview:** Converts the approved post text into semantic HTML and publishes it to LinkedIn with the stored image.
- **Nodes Involved:** 
  - text to html
  - Edit Fields8
  - LinkedIn

- **Node Details:**

  - **text to html**
    - Type: `LangChain Agent`
    - Role: Converts the LinkedIn post markdown into valid, semantic HTML suitable for marketing emails.
    - Configuration:
      - System prompt specifies HTML tags allowed and styling requirements.
      - Embeds the previously uploaded image from S3 at the bottom using an `<img>` tag with the dynamic filename.
    - Output: Raw HTML string.
    - Failures: Formatting errors, timeout, API errors.
  
  - **Edit Fields8**
    - Type: `Set`
    - Role: Sets `output` variable with the HTML content for LinkedIn.
    - Input: From Approval Result1 (approval "Yes") or from revision loop.
  
  - **LinkedIn**
    - Type: `LinkedIn`
    - Role: Publishes the post with embedded image to LinkedIn using OAuth2 credentials.
    - Configuration:
      - Text content from `output`.
      - Person ID specified.
      - Share media category set to IMAGE.
    - Failures: LinkedIn API errors, auth expiry, content size limits.

---

### 3. Summary Table

| Node Name                 | Node Type                           | Functional Role                           | Input Node(s)               | Output Node(s)               | Sticky Note                              |
|---------------------------|-----------------------------------|-----------------------------------------|-----------------------------|-----------------------------|-----------------------------------------|
| When chat message received| chatTrigger (LangChain)            | Receive user input post via chat        | -                           | Edit Fields                 | input post                              |
| Edit Fields               | Set                               | Assign initial post variable             | When chat message received  | AI Agent                    |                                         |
| AI Agent                  | LangChain Agent                   | Clean input, create banner text          | Edit Fields                 | Edit Fields6                | clean post and add image banner         |
| Structured Output Parser  | outputParserStructured (LangChain)| Parse AI output JSON                      | AI Agent                   | AI Agent                   |                                         |
| Edit Fields6              | Set                               | Extract banner lines, create safeName    | AI Agent                   | HTTP Request6              | image kit io for adding text on image https://imagekit.io/ |
| HTTP Request6             | HTTP Request                     | Generate banner image with overlay text  | Edit Fields6               | S32, Merge                 |                                         |
| S32                       | S3                               | Upload image to S3 bucket                 | HTTP Request6              | text to html               | s3 bucket for dynamic display of image  |
| text to html              | LangChain Agent                   | Convert post markdown to semantic HTML   | S32                        | Send Content for Approval1 |                                         |
| Send Content for Approval1| Gmail                            | Send email for human approval             | text to html               | Approval Result1           | approval(human in the loop)              |
| Approval Result1          | Switch                           | Route based on approval response          | Send Content for Approval1 | Edit Fields8, Revision based on feedback |                                         |
| Revision based on feedback| LangChain Agent                   | Revise post text based on feedback        | Approval Result1           | Edit Fields                | revision loop                           |
| Edit Fields8              | Set                               | Set final output HTML                      | Approval Result1           | Merge                      |                                         |
| Merge                     | Merge                            | Combine post and image data                | Edit Fields8, S32          | LinkedIn                   | post to linkedin                        |
| LinkedIn                  | LinkedIn                         | Publish final post to LinkedIn             | Merge                      | -                          |                                         |
| Sticky Note               | Sticky Note                      | Informational notes                        | -                         | -                         | revision loop                           |
| Sticky Note1              | Sticky Note                      | Informational notes                        | -                         | -                         | approval(human in the loop)              |
| Sticky Note2              | Sticky Note                      | Informational notes                        | -                         | -                         | post to linkedin                        |
| Sticky Note3              | Sticky Note                      | Informational notes                        | -                         | -                         | input post                              |
| Sticky Note4              | Sticky Note                      | Informational notes                        | -                         | -                         | clean post and add image banner         |
| Sticky Note5              | Sticky Note                      | Informational notes                        | -                         | -                         | image kit io for adding text on image https://imagekit.io/ |
| Sticky Note6              | Sticky Note                      | Informational notes                        | -                         | -                         | s3 bucket for dynamic display of image  |
| Sticky Note7              | Sticky Note                      | No content                                | -                         | -                         |                                         |
| Sticky Note8              | Sticky Note                      | No content                                | -                         | -                         |                                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Node:**
   - Type: `When chat message received` (LangChain chatTrigger)
   - Configuration:
     - Enable webhook with file upload allowed.
     - No additional parameters.
   - Position: Input start.
  
2. **Create Set Node "Edit Fields":**
   - Type: `Set`
   - Configuration:
     - Assign field `post` = `{{$json.chatInput}}`.
  
3. **Create LangChain Agent "AI Agent":**
   - Type: `LangChain Agent`
   - Configuration:
     - Input text from `post`.
     - System prompt:
       - Clean formatting, preserve emojis.
       - Generate banner text split into `line1` (max 5 words) and `line2` (remaining, max 5 words).
       - URL encode spaces as `%20`.
       - Output as JSON only.
     - Enable output parsing.
  
4. **Create Structured Output Parser:**
   - Type: `outputParserStructured` (LangChain)
   - Configuration:
     - Provide JSON schema example with keys: `postclean`, `line1`, `line2`.
  
5. **Create Set Node "Edit Fields6":**
   - Type: `Set`
   - Configuration:
     - Assign `line1` and `line2` from parsed AI output.
     - Create `safeName`:
       - First word of `line1` (letters/numbers only) + current date in YYYYMMDD format.
  
6. **Create HTTP Request Node "HTTP Request6":**
   - Type: `HTTP Request`
   - Configuration:
     - Method: GET
     - URL:
       ```
       https://ik.imagekit.io/(imagekit image link).png?updatedAt=1751960203232&tr=l-text,i-{{ $json.line1 }},fs-30,co-FFFFFF,ff-Montserrat,tg-b,lx-20,ly-85,l-end:l-text,i-{{ $json.line2 }},fs-30,co-FFFFFF,ff-Montserrat,tg-b,lx-20,ly-130,l-end
       ```
     - Response: Expect image/binary.
  
7. **Create S3 Node "S32":**
   - Type: `S3`
   - Configuration:
     - Operation: Upload.
     - Bucket Name: `rag`.
     - File Name: from `safeName`.
     - Credentials: Configure with valid S3 access.
  
8. **Create LangChain Agent "text to html":**
   - Type: `LangChain Agent`
   - Configuration:
     - Input: `postclean` from AI Agent.
     - System prompt:
       - Convert markdown LinkedIn post to semantic HTML.
       - Use specified tags (`<p>`, `<ul>`, `<li>`, etc.).
       - Minimal inline CSS for readability.
       - Embed image `<img>` with src pointing to S3 URL using `safeName`.
       - Output only raw HTML.
  
9. **Create Gmail Node "Send Content for Approval1":**
   - Type: `Gmail`
   - Configuration:
     - Recipient: dynamic email from input JSON.
     - Subject: "Approval Required for LinkedIn post".
     - Message: includes post HTML.
     - Send operation: `sendAndWait`.
     - Form fields:
       - Dropdown: Approve Content? (Yes, No, Cancel), required.
       - Textarea: Content Feedback.
     - Credentials: Gmail OAuth2.
  
10. **Create Switch Node "Approval Result1":**
    - Type: `Switch`
    - Configuration:
      - Three outputs: Yes, No, Cancel.
      - Conditions based on dropdown selection in approval email.
  
11. **Create LangChain Agent "Revision based on feedback":**
    - Type: `LangChain Agent`
    - Configuration:
      - Prompt:
        - Role: assistant revising LinkedIn post.
        - Inputs: original cleaned post and user feedback.
        - Instructions: minor edits only, preserve tone and structure.
      - Output: revised post text.
  
12. **Create Set Node "Edit Fields8":**
    - Type: `Set`
    - Configuration:
      - Assign `output` = revised post HTML.
  
13. **Create Merge Node "Merge":**
    - Type: `Merge`
    - Configuration:
      - Mode: Combine all inputs.
      - Inputs: S3 upload output and final post output.
  
14. **Create LinkedIn Node "LinkedIn":**
    - Type: `LinkedIn`
    - Configuration:
      - Text: from merged output.
      - Person ID: set to appropriate user.
      - Share media category: IMAGE.
      - Credentials: LinkedIn OAuth2.
  
15. **Connect all nodes in the order described in the workflow connections, ensuring loops back from revision to Edit Fields for reprocessing, and human approval controlling flow.**

---

### 5. General Notes & Resources

| Note Content                                                                                                         | Context or Link                            |
|----------------------------------------------------------------------------------------------------------------------|--------------------------------------------|
| ImageKit is used for dynamic text overlay on images with URL-based transformations.                                  | https://imagekit.io/                        |
| The S3 bucket named `rag` stores generated images for reuse and dynamic referencing in posts.                        | S3 storage node configuration              |
| The workflow implements a human-in-the-loop approval process via Gmail with custom form responses for feedback.      | Gmail node with sendAndWait and form fields|
| LinkedIn publishing requires OAuth2 credentials and person ID configuration.                                         | LinkedIn node OAuth2 setup                   |

---

**Disclaimer:**  
The text provided is generated exclusively from an automated n8n workflow. All processing complies with current content policies and contains no illegal, offensive, or protected content. All data handled is legal and public.