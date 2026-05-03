Convert YouTube Videos into SEO Blog Posts with GPT-4o, Dumpling AI, and Flux

https://n8nworkflows.xyz/workflows/convert-youtube-videos-into-seo-blog-posts-with-gpt-4o--dumpling-ai--and-flux-3531


# Convert YouTube Videos into SEO Blog Posts with GPT-4o, Dumpling AI, and Flux

### 1. Workflow Overview

This workflow automates the conversion of a YouTube video into a fully formatted, SEO-optimized blog post, complete with a relevant AI-generated image, and sends the final package via email. It is designed for content creators who want to repurpose video content efficiently into written blog posts ready for publication or further editing.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Manual trigger and variable setup for YouTube video URL and recipient email.
- **1.2 Transcript Retrieval:** Fetch the transcript of the specified YouTube video using Dumpling AI.
- **1.3 Blog Post Generation:** Use OpenAI’s GPT-4o model to generate a detailed SEO blog post based on the transcript.
- **1.4 Image Generation:** Generate a blog post image using Dumpling AI’s FLUX.1-dev model based on the AI-generated image prompt.
- **1.5 Content Formatting:** Convert the blog post content from Markdown to HTML.
- **1.6 Image Download:** Download the generated image to attach to the email.
- **1.7 Email Dispatch:** Send the blog post (HTML content) and image as an email to the specified recipient.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block initiates the workflow manually and sets the key input variables: the YouTube video URL and the recipient’s email address.

- **Nodes Involved:**  
  - When clicking ‘Test workflow’ (Manual Trigger)  
  - Set Variables  
  - Sticky Note (Set Variables instructions)

- **Node Details:**

  - **When clicking ‘Test workflow’**  
    - Type: Manual Trigger  
    - Role: Starts the workflow on user command.  
    - Configuration: Default manual trigger, no parameters.  
    - Inputs: None  
    - Outputs: Connects to "Set Variables" node.  
    - Edge Cases: None specific; user must trigger manually.

  - **Set Variables**  
    - Type: Set  
    - Role: Defines workflow input variables.  
    - Configuration:  
      - `YouTube Video Url`: Set to "https://www.youtube.com/watch?v=Dpie2Cd4iB4" (example URL).  
      - `Recipient Email Address`: Set to "example@example.com" (example email).  
    - Inputs: From manual trigger  
    - Outputs: Connects to "Get YouTube Transcript" node.  
    - Edge Cases: Variables must be valid; invalid URLs or emails will cause downstream failures.

  - **Sticky Note (Set Variables)**  
    - Type: Sticky Note  
    - Role: Provides instructions on setting variables.  
    - Content: Explains the purpose of the variables and what to set.

---

#### 2.2 Transcript Retrieval

- **Overview:**  
  Retrieves the transcript of the specified YouTube video using Dumpling AI’s API. The video must have captions or subtitles enabled.

- **Nodes Involved:**  
  - Get YouTube Transcript  
  - Sticky Note1 (Transcript retrieval explanation)

- **Node Details:**

  - **Get YouTube Transcript**  
    - Type: HTTP Request  
    - Role: Calls Dumpling AI API to fetch the transcript.  
    - Configuration:  
      - Method: POST  
      - URL: `https://app.dumplingai.com/api/v1/get-youtube-transcript`  
      - Body Parameters:  
        - `videoUrl`: Expression pulling from `YouTube Video Url` variable.  
        - `includeTimestamps`: false (transcript without timestamps).  
      - Authentication: HTTP Bearer Token and HTTP Header Auth (Dumpling AI credentials).  
    - Inputs: From "Set Variables"  
    - Outputs: Connects to "Generate Blog Post" node.  
    - Edge Cases:  
      - Failure if video has no captions/subtitles.  
      - API rate limits or auth errors.  
      - Network timeouts.

  - **Sticky Note1**  
    - Type: Sticky Note  
    - Role: Explains the transcript retrieval step and prerequisites (captions enabled).

---

#### 2.3 Blog Post Generation

- **Overview:**  
  Uses OpenAI’s GPT-4o model to generate a detailed, SEO-optimized blog post based on the transcript. The output includes title, description, blog image prompt, and full blog content in Markdown.

- **Nodes Involved:**  
  - Generate Blog Post  
  - Sticky Note2 (Blog post generation explanation)

- **Node Details:**

  - **Generate Blog Post**  
    - Type: OpenAI (Langchain) Node  
    - Role: Generates blog post content from transcript.  
    - Configuration:  
      - Model: GPT-4o  
      - System prompt: Detailed instructions to analyze transcript, extract key points, perform SEO keyword research, structure blog post with headings, and output JSON with fields: title, blogImagePrompt, description, content.  
      - Input: Transcript from previous node.  
      - Output: JSON with blog post details.  
      - Credentials: OpenAI API key.  
    - Inputs: From "Get YouTube Transcript"  
    - Outputs: Connects to "Generate AI Image" node.  
    - Edge Cases:  
      - API rate limits or auth errors.  
      - Model response errors or invalid JSON output.  
      - Transcript too short or irrelevant content may produce poor blog posts.

  - **Sticky Note2**  
    - Type: Sticky Note  
    - Role: Describes the blog post generation step and suggests improvements like section-by-section generation or AI Agent integration.

---

#### 2.4 Image Generation

- **Overview:**  
  Generates an AI image for the blog post using Dumpling AI’s FLUX.1-dev model, based on the image prompt generated by the blog post node.

- **Nodes Involved:**  
  - Generate AI Image  
  - Sticky Note3 (Image generation explanation)

- **Node Details:**

  - **Generate AI Image**  
    - Type: HTTP Request  
    - Role: Calls Dumpling AI API to generate an image from the prompt.  
    - Configuration:  
      - Method: POST  
      - URL: `https://app.dumplingai.com/api/v1/generate-ai-image`  
      - JSON Body: Includes model "FLUX.1-dev" and prompt from blogImagePrompt field in previous node’s output.  
      - Authentication: HTTP Bearer Token and HTTP Header Auth (Dumpling AI credentials).  
    - Inputs: From "Generate Blog Post"  
    - Outputs: Connects to "Markdown" node.  
    - Edge Cases:  
      - API errors or auth failures.  
      - Image generation may fail or produce irrelevant images if prompt is poor.  
      - Temporary image URLs require downloading before use.

  - **Sticky Note3**  
    - Type: Sticky Note  
    - Role: Explains the image generation step and notes about model choice and limitations (e.g., text in images).

---

#### 2.5 Content Formatting

- **Overview:**  
  Converts the blog post content from Markdown (produced by OpenAI) into HTML for proper email formatting.

- **Nodes Involved:**  
  - Markdown  
  - Sticky Note4 (Markdown to HTML explanation)

- **Node Details:**

  - **Markdown**  
    - Type: Markdown  
    - Role: Converts Markdown blog content to HTML.  
    - Configuration:  
      - Mode: markdownToHtml  
      - Input: `content` field from blog post JSON.  
      - Output Key: `htmlContent`  
    - Inputs: From "Generate AI Image"  
    - Outputs: Connects to "Download Image" node.  
    - Edge Cases:  
      - Invalid Markdown may cause conversion issues.  
      - Large content may increase processing time.

  - **Sticky Note4**  
    - Type: Sticky Note  
    - Role: Explains the need to convert Markdown to HTML for email formatting.

---

#### 2.6 Image Download

- **Overview:**  
  Downloads the AI-generated image from the temporary URL to attach it to the outgoing email.

- **Nodes Involved:**  
  - Download Image  
  - Sticky Note5 (Image download explanation)

- **Node Details:**

  - **Download Image**  
    - Type: HTTP Request  
    - Role: Downloads the image file from the URL provided by Dumpling AI.  
    - Configuration:  
      - Method: GET (default)  
      - URL: Extracted from the first image URL in the "Generate AI Image" node output.  
    - Inputs: From "Markdown"  
    - Outputs: Connects to "Gmail" node.  
    - Edge Cases:  
      - Temporary URLs may expire quickly.  
      - Network errors or invalid URLs cause failure.

  - **Sticky Note5**  
    - Type: Sticky Note  
    - Role: Explains the necessity of downloading the image to attach it to the email.

---

#### 2.7 Email Dispatch

- **Overview:**  
  Sends the final blog post (HTML content) and the AI-generated image as an email to the specified recipient.

- **Nodes Involved:**  
  - Gmail  
  - Sticky Note6 (Email sending explanation)

- **Node Details:**

  - **Gmail**  
    - Type: Gmail  
    - Role: Sends an email with blog post content and image attachment.  
    - Configuration:  
      - Recipient: From `Recipient Email Address` variable.  
      - Subject: Blog post title from AI output.  
      - Message Body: Includes description and HTML content of the blog post.  
      - Attachments: Binary image downloaded in previous node.  
      - Credentials: Gmail OAuth2 credentials.  
    - Inputs: From "Download Image"  
    - Outputs: None (end of workflow)  
    - Edge Cases:  
      - Email sending failures due to auth or quota limits.  
      - Large attachments may cause issues.  
      - Invalid email addresses cause delivery failure.

  - **Sticky Note6**  
    - Type: Sticky Note  
    - Role: Explains that the email contains all generated content for record and publication preparation.

---

### 3. Summary Table

| Node Name                  | Node Type                  | Functional Role                              | Input Node(s)               | Output Node(s)             | Sticky Note                                                                                              |
|----------------------------|----------------------------|----------------------------------------------|-----------------------------|----------------------------|--------------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger             | Starts workflow manually                      | None                        | Set Variables              |                                                                                                        |
| Set Variables              | Set                        | Defines YouTube URL and recipient email      | When clicking ‘Test workflow’ | Get YouTube Transcript     | Set your variables here, such as YouTube Video URL and Recipient Email Address.                         |
| Sticky Note                | Sticky Note                | Instructions for setting variables           | None                        | None                       | Set your variables here, such as: YouTube Video URL and Recipient Email Address.                        |
| Get YouTube Transcript     | HTTP Request               | Fetches YouTube video transcript via Dumpling AI | Set Variables               | Generate Blog Post         | This step gets the transcript of the YouTube video with Dumpling AI. The video must have captions enabled. |
| Sticky Note1               | Sticky Note                | Explains transcript retrieval step            | None                        | None                       | This step gets the transcript of the YouTube video with Dumpling AI. The target video must have captions or subtitles enabled. |
| Generate Blog Post         | OpenAI (Langchain)         | Generates SEO blog post JSON from transcript | Get YouTube Transcript      | Generate AI Image          | Here we get GPT-4o to generate a detailed SEO blog post including title, description, image prompt, and content. |
| Sticky Note2               | Sticky Note                | Explains blog post generation                  | None                        | None                       | Here we get GPT-4o (or a model of your choice) to generate a detailed SEO blog post.                   |
| Generate AI Image          | HTTP Request               | Generates blog post image via Dumpling AI    | Generate Blog Post          | Markdown                   | Here we use the FLUX.1-dev model via Dumpling AI to generate an image for the blog post.               |
| Sticky Note3               | Sticky Note                | Explains image generation step                 | None                        | None                       | Here we use the FLUX.1-dev model via Dumpling AI to generate a image for the blog post.                |
| Markdown                   | Markdown                   | Converts blog content from Markdown to HTML  | Generate AI Image           | Download Image             | OpenAI LLMs tend to output markdown. We need to convert to HTML for formatting in the Gmail node.      |
| Sticky Note4               | Sticky Note                | Explains Markdown to HTML conversion           | None                        | None                       | OpenAI LLMs tend to output markdown. We need to convert to HTML for formatting in the Gmail node.      |
| Download Image             | HTTP Request               | Downloads AI-generated image for email attachment | Markdown                   | Gmail                      | The image URL is temporary, so we need to download the image and attach it to the email being sent.    |
| Sticky Note5               | Sticky Note                | Explains image download necessity              | None                        | None                       | The image URL is a temporary URL, so we need to download the image and attach it to the email being sent. |
| Gmail                     | Gmail                      | Sends email with blog post and image          | Download Image              | None                       | We send all generated content to your email address so you have a record of all generations and can get it ready for publication. |
| Sticky Note6               | Sticky Note                | Explains email sending step                      | None                        | None                       | We send all generated content to your email address so you have a record of all generations and can get it ready for publication. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Add a "Manual Trigger" node named "When clicking ‘Test workflow’".

2. **Create Set Variables Node**  
   - Add a "Set" node named "Set Variables".  
   - Add two string fields:  
     - `YouTube Video Url` with a sample YouTube URL (e.g., `https://www.youtube.com/watch?v=Dpie2Cd4iB4`).  
     - `Recipient Email Address` with a sample email (e.g., `example@example.com`).  
   - Connect "When clicking ‘Test workflow’" → "Set Variables".

3. **Create HTTP Request Node for Transcript**  
   - Add an "HTTP Request" node named "Get YouTube Transcript".  
   - Set Method to POST.  
   - URL: `https://app.dumplingai.com/api/v1/get-youtube-transcript`.  
   - Body Parameters (JSON or form-data):  
     - `videoUrl`: Expression `={{ $json["YouTube Video Url"] }}`  
     - `includeTimestamps`: false  
   - Authentication: Use HTTP Bearer Auth and HTTP Header Auth with Dumpling AI credentials.  
   - Connect "Set Variables" → "Get YouTube Transcript".

4. **Create OpenAI Node for Blog Post Generation**  
   - Add an OpenAI node (Langchain or OpenAI node) named "Generate Blog Post".  
   - Select model `gpt-4o`.  
   - Set system prompt with detailed instructions to analyze transcript and output JSON with fields: title, blogImagePrompt, description, content.  
   - Input message content: transcript from previous node (`{{ $json.transcript }}`).  
   - Enable JSON output parsing.  
   - Connect "Get YouTube Transcript" → "Generate Blog Post".  
   - Configure OpenAI API credentials.

5. **Create HTTP Request Node for AI Image Generation**  
   - Add an "HTTP Request" node named "Generate AI Image".  
   - Method: POST  
   - URL: `https://app.dumplingai.com/api/v1/generate-ai-image`  
   - Body (JSON):  
     ```json
     {
       "model": "FLUX.1-dev",
       "input": {
         "prompt": "{{ $json.message.content.blogImagePrompt }}"
       }
     }
     ```  
   - Authentication: Use Dumpling AI credentials (Bearer and Header Auth).  
   - Connect "Generate Blog Post" → "Generate AI Image".

6. **Create Markdown Node to Convert Content**  
   - Add a "Markdown" node named "Markdown".  
   - Mode: markdownToHtml  
   - Input: `{{ $json.message.content.content }}` (blog post content)  
   - Output key: `htmlContent`  
   - Connect "Generate AI Image" → "Markdown".

7. **Create HTTP Request Node to Download Image**  
   - Add an "HTTP Request" node named "Download Image".  
   - Method: GET (default)  
   - URL: `{{ $json.images[0].url }}` (first image URL from AI image generation output)  
   - Connect "Markdown" → "Download Image".

8. **Create Gmail Node to Send Email**  
   - Add a "Gmail" node named "Gmail".  
   - Set "Send To": `{{ $('Set Variables').item.json['Recipient Email Address'] }}`  
   - Subject: `{{ $('Generate Blog Post').item.json.message.content.title }}`  
   - Message:  
     ```
     Description: {{ $('Generate Blog Post').item.json.message.content.description }}

     Content:
     {{ $('Markdown').item.json.htmlContent }}
     ```  
   - Attachments: Attach binary data from "Download Image" node.  
   - Connect "Download Image" → "Gmail".  
   - Configure Gmail OAuth2 credentials.

9. **Add Sticky Notes (Optional but Recommended)**  
   - Add sticky notes near each logical block with the explanations as per the original workflow for clarity.

10. **Test Workflow**  
    - Trigger manually and verify each step completes successfully.  
    - Check email for correct blog post content and image attachment.

---

### 5. General Notes & Resources

| Note Content                                                                                                   | Context or Link                                                                                   |
|---------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Workflow uses GPT-4o model from OpenAI for blog post generation.                                               | OpenAI platform: https://platform.openai.com/                                                  |
| Dumpling AI provides transcript retrieval and AI image generation services.                                   | Dumpling AI: https://www.dumplingai.com/                                                       |
| Dumpling AI offers 250 free credits initially; check pricing for extended use.                                | Pricing: https://www.dumplingai.com/pricing                                                    |
| Email sending uses Gmail OAuth2 credentials; ensure proper authentication setup.                              | Gmail OAuth2 setup documentation in n8n docs                                                    |
| To improve, consider direct CMS publishing, AI Agents for content generation, or batch processing.            | Suggestions included in workflow description                                                    |
| Temporary image URLs require downloading before attaching to emails to avoid broken links.                    | Explained in Sticky Note5                                                                        |
| OpenAI LLMs output Markdown by default; conversion to HTML is necessary for email formatting.                 | Explained in Sticky Note4                                                                        |
| Workflow requires YouTube videos with captions/subtitles enabled for transcript extraction.                   | Explained in Sticky Note1                                                                        |

---

This documentation provides a detailed, structured reference for understanding, reproducing, and modifying the "Convert YouTube Videos into SEO Blog Posts" workflow, ensuring robust integration and error anticipation.