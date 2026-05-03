Generate LinkedIn Posts and AI Images from Web Pages with Airtop and GPT-4

https://n8nworkflows.xyz/workflows/generate-linkedin-posts-and-ai-images-from-web-pages-with-airtop-and-gpt-4-5774


# Generate LinkedIn Posts and AI Images from Web Pages with Airtop and GPT-4

### 1. Workflow Overview

This workflow automates the generation of LinkedIn posts and corresponding AI-generated images based on any given web page URL. It targets content creators and marketers who want to quickly transform web content (e.g., blog posts, case studies, product updates) into polished LinkedIn posts with branded visuals. The workflow integrates web scraping, AI text generation, AI image prompt creation, image rendering via a sub-workflow, and Slack-based review and feedback loops for iterative refinements.

The workflow is logically divided into three main blocks:

- **1.1 Input Reception & Web Scraping**: Receives the user input (web page URL and optional instructions) and scrapes the web page content using Airtop.

- **1.2 Content & Image Generation**: Generates the LinkedIn post text using GPT-4 based on the scraped content and user instructions; then creates an image prompt aligned with the post for visual generation.

- **1.3 Review, Revision & Finalization**: Sends the post and image prompt to Slack for human review; collects feedback; processes revision requests or approvals; generates the final branded image via a sub-workflow; and posts the approved final content back to Slack.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Web Scraping

**Overview**  
This block handles the initial user input via a form, capturing the URL and optional instructions, and then scrapes the web page content using Airtop’s extraction API.

**Nodes Involved**  
- On form submission  
- Get Page  
- Set Params

**Node Details**

- **On form submission**  
  - Type: Form Trigger  
  - Role: Entry point capturing user input for URL and instructions.  
  - Config: Form titled "LinkedIn Post Generator" with fields for "Page URL" (required) and "Instructions" (optional textarea).  
  - Inputs: HTTP/webhook trigger via form.  
  - Outputs: Passes user input JSON downstream.  
  - Edge Cases: Missing or malformed URL input; form submission failures; webhook timeout.  

- **Get Page**  
  - Type: Airtop Node (Web Scraping)  
  - Role: Scrape content from the provided URL.  
  - Config: Uses the "scrape" operation with a new session, URL dynamically from form input (`Page URL`).  
  - Credentials: Airtop API key required.  
  - Inputs: JSON with `Page URL`.  
  - Outputs: Scraped webpage content in `data.modelResponse.scrapedContent.text`.  
  - Edge Cases: URL unreachable, scraping fails, Airtop API errors, rate limiting, empty content.  

- **Set Params**  
  - Type: Set Node  
  - Role: Prepares variables for downstream nodes, including URL, instructions, initial Post placeholder, and scraped content.  
  - Config: Assigns URL and instructions from form submission; stores scraped content under `=Webpage_content`.  
  - Inputs: Data from "Get Page".  
  - Outputs: Sets structured JSON for post generation.  
  - Edge Cases: Missing or empty scraped content; instructions field empty or undefined.

---

#### 2.2 Content & Image Generation

**Overview**  
Generates an educational and engaging LinkedIn post from the scraped content and user instructions using GPT-4; then creates an AI image prompt tailored to the post for marketing-style visuals.

**Nodes Involved**  
- Generate Post Text  
- Set Post  
- Set Post1  
- OpenAI Chat Model  
- Check Post Text Feedback  
- Set Image Prompt  
- Image Prompt Generator  
- Set Image Prompt1

**Node Details**

- **Generate Post Text**  
  - Type: LangChain OpenAI (Text Completion)  
  - Role: Generate LinkedIn post text using GPT-4 (as assistant).  
  - Config: Prompt combines user instructions, initial post placeholder, and scraped page content; instructs GPT-4 to write helpful, educational, engaging LinkedIn posts.  
  - Credentials: OpenAI API key (Airtop Email Writer).  
  - Inputs: From "Set Params".  
  - Outputs: AI-generated post text.  
  - Edge Cases: API rate limits, content generation failures, prompt misinterpretation.  

- **Set Post**  
  - Type: Set Node  
  - Role: Stores URL, instructions, and generated post text for review.  
  - Inputs: From "Generate Post Text".  
  - Outputs: JSON with URL, instructions, and post.  

- **Set Post1**  
  - Type: Set Node  
  - Role: Updates post and instructions after feedback classification.  
  - Inputs: From "Check Post Text Feedback".  
  - Outputs: Updated post and instructions for re-generation or finalization.  

- **OpenAI Chat Model**  
  - Type: LangChain OpenAI Chat (GPT-4)  
  - Role: Used for classifying text feedback from reviewers (human validation on Slack).  
  - Inputs: Reviewer feedback text.  
  - Outputs: Classifies feedback into “Edit”, “Approved”, or “Declined”.  
  - Edge Cases: Misclassification of feedback, API errors.  

- **Check Post Text Feedback**  
  - Type: Text Classifier (LangChain)  
  - Role: Classifies human feedback on post text to determine next action (edit, approve, decline).  
  - Inputs: Slack human feedback on post text.  
  - Outputs: Triggers re-generation or approval flow.  

- **Set Image Prompt**  
  - Type: Set Node  
  - Role: Initializes or resets image prompt and carries forward post text for image prompt generation.  
  - Inputs: Form submission or post text.  
  - Outputs: JSON with empty or existing image prompt and post text.  

- **Image Prompt Generator**  
  - Type: LangChain OpenAI (Text Completion)  
  - Role: Generates an AI image prompt from the LinkedIn post text aimed at marketing visuals (not literal photos).  
  - Config: Detailed prompt guides AI to create brand-aligned, polished, modern, marketing-style graphic prompts suitable for AI image generation tools.  
  - Credentials: OpenAI API key (Airtop Email Writer).  
  - Inputs: Post text and optionally previous image prompt and user instructions.  
  - Outputs: Image prompt string.  
  - Edge Cases: Ambiguous or insufficient input text leading to poor image prompt quality.  

- **Set Image Prompt1**  
  - Type: Set Node  
  - Role: Stores generated image prompt and resets image ID placeholder for downstream image generation.  
  - Inputs: From "Image Prompt Generator".  
  - Outputs: JSON with `Image_prompt` and empty `Image_id`.  

---

#### 2.3 Review, Revision & Finalization

**Overview**  
Sends generated posts and images to Slack for human review, collects feedback, processes revision requests or approval, invokes sub-workflows for image generation, and finally posts approved content back to Slack.

**Nodes Involved**  
- Human Revision of Text  
- Human Review of Visual  
- Check Image Feedback  
- Set Image Prompt2  
- Generate on-brand image  
- Final Version  
- Generate Image

**Node Details**

- **Human Revision of Text**  
  - Type: Slack Node (Send and Wait for Response)  
  - Role: Sends generated LinkedIn post text to Slack channel for human feedback with a response form titled "Image Feedback".  
  - Config: Message includes post text and URL; waits up to 3 days for feedback.  
  - Inputs: From "Set Post".  
  - Outputs: Human free-text feedback on post text.  
  - Credentials: Slack OAuth2.  
  - Edge Cases: Slack API failures, no feedback received, invalid responses.  

- **Human Review of Visual**  
  - Type: Slack Node (Send and Wait for Response)  
  - Role: Sends post, image prompt, and generated image URL to Slack for human visual feedback with a response form titled "Image Feedback".  
  - Inputs: From "Generate on-brand image" and "Set Image Prompt1".  
  - Outputs: Human free-text feedback on image prompt and image.  
  - Credentials: Slack OAuth2.  
  - Edge Cases: Slack API failures, no feedback, ambiguous feedback.  

- **Check Image Feedback**  
  - Type: Text Classifier (LangChain)  
  - Role: Classifies Slack feedback on image prompt/image into categories Edit, Approved, Declined.  
  - Inputs: Human feedback text from Slack.  
  - Outputs: Branches flow into revision (Set Image Prompt2) or finalization (Final Version).  

- **Set Image Prompt2**  
  - Type: Set Node  
  - Role: Prepares revised image prompt including original prompt and user revision instructions; sets Image ID for regeneration.  
  - Inputs: From "Check Image Feedback".  
  - Outputs: Updated `Image_prompt` and `Image_id` for image generation.  

- **Generate on-brand image**  
  - Type: Execute Workflow (Sub-Workflow)  
  - Role: Invokes a sub-workflow dedicated to generating and editing branded images based on the prompt and image ID.  
  - Inputs: Receives prompt and image drive ID.  
  - Outputs: Returns branded image URL and unbranded image ID.  
  - Edge Cases: Sub-workflow failures, timeout, invalid prompt.  

- **Final Version**  
  - Type: Slack Node (Message Post)  
  - Role: Sends final approved post text and branded image URL to Slack channel for publishing or final review.  
  - Inputs: From "Generate on-brand image" and "Set Image Prompt".  
  - Credentials: Slack OAuth2.  
  - Edge Cases: Slack API rate limits or message failures.  

- **Generate Image**  
  - Type: Form Trigger  
  - Role: Alternative entry point to generate or revise image prompts and visuals when post text already exists.  
  - Config: Form with a required "Post" textarea field.  
  - Outputs: Starts image prompt generation and revision flow.  

---

### 3. Summary Table

| Node Name              | Node Type                            | Functional Role                            | Input Node(s)                  | Output Node(s)                   | Sticky Note                                                                                   |
|------------------------|------------------------------------|--------------------------------------------|-------------------------------|---------------------------------|----------------------------------------------------------------------------------------------|
| On form submission     | Form Trigger                       | Entry point for user input (URL, instructions) | -                             | Get Page                        | # To generate post and visual - start here!                                                  |
| Get Page               | Airtop                            | Scrape webpage content from URL             | On form submission            | Set Params                     |                                                                                              |
| Set Params             | Set                              | Prepares variables for post generation       | Get Page                     | Generate Post Text             |                                                                                              |
| Generate Post Text     | LangChain OpenAI (Text)           | Generate educational, engaging LinkedIn post | Set Params                   | Set Post                      | # Content Generation & Revisions                                                             |
| Set Post               | Set                              | Store post text and metadata for review      | Generate Post Text           | Human Revision of Text         |                                                                                              |
| Human Revision of Text | Slack                            | Sends post text to Slack, waits for feedback | Set Post                    | Check Post Text Feedback       |                                                                                              |
| Check Post Text Feedback| LangChain Text Classifier         | Classify human feedback on post text         | Human Revision of Text       | Set Post1, Set Image Prompt    |                                                                                              |
| Set Post1              | Set                              | Update post text after review                  | Check Post Text Feedback     | Generate Post Text             |                                                                                              |
| Set Image Prompt       | Set                              | Initialize/reset image prompt and carry post  | Generate Image, Check Post Text Feedback | Image Prompt Generator     |                                                                                              |
| Image Prompt Generator | LangChain OpenAI (Text)           | Generate AI image prompt from LinkedIn post   | Set Image Prompt             | Set Image Prompt1              | # Image Generation & Revisions                                                               |
| Set Image Prompt1      | Set                              | Store generated image prompt and reset image ID | Image Prompt Generator       | Generate on-brand image        |                                                                                              |
| Generate on-brand image| Execute Workflow (Sub-Workflow)   | Generate branded AI image based on prompt     | Set Image Prompt1            | Human Review of Visual         |                                                                                              |
| Human Review of Visual | Slack                            | Sends image prompt and visual to Slack for feedback | Generate on-brand image      | Check Image Feedback           |                                                                                              |
| Check Image Feedback   | LangChain Text Classifier         | Classify human feedback on image prompt/image | Human Review of Visual       | Set Image Prompt2, Final Version |                                                                                              |
| Set Image Prompt2      | Set                              | Prepare revised image prompt and image ID     | Check Image Feedback         | Generate on-brand image        |                                                                                              |
| Final Version          | Slack                            | Posts final approved post and image to Slack  | Check Image Feedback         | -                             | # Send Final Post and Visual                                                                 |
| Generate Image         | Form Trigger                     | Alternate entry for image prompt generation    | -                           | Set Image Prompt               | # To generate just a branded visual - start here!                                            |
| OpenAI Chat Model      | LangChain OpenAI Chat (GPT-4)     | Classify feedback text for post revisions      | Human Revision of Text       | Check Post Text Feedback       |                                                                                              |
| OpenAI Chat Model1     | LangChain OpenAI Chat (GPT-4)     | Classify feedback text for image revisions     | Human Review of Visual       | Check Image Feedback           |                                                                                              |
| Set Post1              | Set                              | Update post text after review                    | Check Post Text Feedback     | Generate Post Text             |                                                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Form Trigger Node: "On form submission"**  
   - Type: `Form Trigger`  
   - Configure form title: "LinkedIn Post Generator"  
   - Fields:  
     - "Page URL" (required, single line text, placeholder: "https://www.airtop.ai/automations/ai-web-agent-n8n")  
     - "Instructions" (optional, textarea)  
   - Description: "Fill out these fields and you'll have a full LinkedIn post ready to go in a minute."  

2. **Create Airtop Node: "Get Page"**  
   - Type: `Airtop`  
   - Operation: "scrape" with "new" session mode  
   - URL: Use expression `{{$json["Page URL"]}}` from form trigger  
   - Credentials: Airtop API key  

3. **Create Set Node: "Set Params"**  
   - Type: `Set`  
   - Assign variables:  
     - URL: `={{ $('On form submission').item.json['Page URL'] }}`  
     - Instructions: `={{ $('On form submission').item.json.Instructions || '' }}` (empty string fallback)  
     - Post: `= - ` (placeholder)  
     - Webpage_content: `={{ $json.data.modelResponse.scrapedContent.text }}` (from Airtop scrape)  

4. **Create LangChain OpenAI Node: "Generate Post Text"**  
   - Type: `LangChain OpenAI` (text completion)  
   - Model: Select GPT-4 (e.g., "gpt-4.1")  
   - Prompt template:  
     ```
     Write a helpful educational, and engaging LinkedIn posts based on a webpage scraped content below or the Post text below and according to the instructions provided by the user.

     Instructions: {{ $json.Instructions }}

     {{ $json.Post }}

     {{ $json.Webpage_content }}
     ```  
   - Credentials: OpenAI API key for post generation  

5. **Create Set Node: "Set Post"**  
   - Assignments:  
     - URL: `={{ $('Set Params').item.json.URL }}`  
     - Instructions: `=-` or as needed  
     - Post: `={{ $json.output }}` (output from post text generation)  

6. **Create Slack Node: "Human Revision of Text"**  
   - Type: `Slack` (Send and Wait for Response)  
   - Message: Include post text and URL for review  
   - Response form title: "Image Feedback"  
   - Timeout: 3 days  
   - Credentials: Slack OAuth2 API  
   - Channel: Target Slack channel for review  

7. **Create LangChain Text Classifier Node: "Check Post Text Feedback"**  
   - Input Text: Feedback from Slack response  
   - Categories: Edit, Approved, Declined (with detailed descriptions)  

8. **Create Set Node: "Set Post1"**  
   - Update post text or instructions based on classifier output for reprocessing  

9. **Create Set Node: "Set Image Prompt"**  
   - Initialize image prompt and carry post text forward:  
     - Image_prompt: empty string or space  
     - Post: use current post text  

10. **Create LangChain OpenAI Node: "Image Prompt Generator"**  
    - Model: GPT-4  
    - Prompt: Detailed instructions to generate marketing-style AI image prompts referencing the LinkedIn post.  
    - Credentials: OpenAI API key for image prompt generation  

11. **Create Set Node: "Set Image Prompt1"**  
    - Assignments:  
      - Image_prompt: output from Image Prompt Generator  
      - Image_id: empty string placeholder  

12. **Create Execute Workflow Node: "Generate on-brand image"**  
    - Mode: "each" with waitForSubWorkflow enabled  
    - Workflow ID: Sub-workflow that generates and edits branded images (must be created separately)  
    - Workflow Inputs:  
      - Prompt: `={{ $json.Image_prompt }}`  
      - Image_drive_id: `={{ $json.Image_id }}`  

13. **Create Slack Node: "Human Review of Visual"**  
    - Sends post, image prompt, and generated image URL to Slack for visual feedback  
    - Response form titled "Image Feedback"  
    - Timeout: 3 days  
    - Credentials: Slack OAuth2 API  

14. **Create LangChain Text Classifier Node: "Check Image Feedback"**  
    - Classifies Slack feedback on image prompt/image into Edit, Approved, Declined  

15. **Create Set Node: "Set Image Prompt2"**  
    - Combines original image prompt, user revision instructions, and sets Image_id from unbranded image for regeneration  

16. **Connect "Set Image Prompt2" to "Generate on-brand image"**  
    - For processing image revisions  

17. **Connect "Check Image Feedback" Approved branch to Slack Node: "Final Version"**  
    - Posts final approved post text and branded image URL back to Slack for publishing  

18. **Create Form Trigger Node: "Generate Image"** (optional alternative entry)  
    - Form titled "Generate Image" with required "Post" textarea  
    - Start flow to regenerate or revise image prompt and visual  

---

### 5. General Notes & Resources

| Note Content  | Context or Link  |
|---------------|------------------|
| **README**: This workflow turns any web page URL into a LinkedIn post with an AI-generated branded image, including Slack-based review and revision cycles. Ideal for marketers and content teams. | See Sticky Note1 in workflow for full README content. |
| Requires Airtop API key for web scraping integration. | Airtop Official Org credential node in workflow. |
| OpenAI API credentials needed twice: once for post generation and once for image prompt generation (both use GPT-4). | OpenAi account 3 and OpenAi Airtop Email Writer credentials used. |
| Slack OAuth2 integration is necessary for sending messages and receiving feedback in a Slack channel. | Slack account 3 credentials used in Slack nodes. |
| Sub-workflow for image generation ("AIRTOP — Generate and Edit On-Brand Images - V3") must be implemented separately with inputs for prompt and image ID. | Execute Workflow node references external workflow ID. |
| To extend: Add LinkedIn publishing node for direct post automation. | Suggested in README sticky note. |
| For branding consistency, image prompts are carefully crafted to be marketing-focused, not literal photos. | See Image Prompt Generator node prompt details. |
| Feedback classification uses AI to automate response handling and streamline revision workflow. | Check Post Text Feedback and Check Image Feedback nodes. |

---

This documentation captures the full structure, logic, and integration points of the "Generate LinkedIn Posts and AI Images from Web Pages with Airtop and GPT-4" workflow, enabling efficient reproduction, modification, and troubleshooting.