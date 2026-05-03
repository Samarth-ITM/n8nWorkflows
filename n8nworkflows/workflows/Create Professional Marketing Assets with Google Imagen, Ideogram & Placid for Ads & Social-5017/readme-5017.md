Create Professional Marketing Assets with Google Imagen, Ideogram & Placid for Ads & Social

https://n8nworkflows.xyz/workflows/create-professional-marketing-assets-with-google-imagen--ideogram---placid-for-ads---social-5017


# Create Professional Marketing Assets with Google Imagen, Ideogram & Placid for Ads & Social

### 1. Workflow Overview

This workflow automates the creation of professional marketing assets for Meta Ads, email campaigns, and social media by integrating AI-powered image generation, copywriting, and templating services. It is designed to take product-related input (including uploaded images), generate styled background images and marketing copy using AI models, then enhance and compose these assets into finished images using Fal.ai and Placid.app. The final images are stored on Google Drive for easy access and distribution.

**Target Use Cases:**  
- Marketing teams or agencies generating multiple ad creatives efficiently  
- Automated content generation for social platforms and email campaigns  
- Combining AI image generation with copywriting and templating for branded assets

**Logical Blocks:**  
- **1.1 Input Reception & Preparation:** Handles form submission and image upload preprocessing  
- **1.2 AI Copywriting & Background Image Prompt Generation:** Generates marketing copy and background image prompts via OpenAI-powered agents  
- **1.3 Background Image Generation & Processing via Fal.ai:** Requests image generation, monitors status, applies background replacement and relighting effects  
- **1.4 Image Composition with Placid.app:** Composes final marketing images using templated design and generated assets  
- **1.5 Asset Download & Storage:** Downloads final images and saves them to Google Drive

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Preparation

**Overview:**  
This block triggers the workflow upon form submission, obtains a temporary URL for the uploaded image, and prepares the input for AI processing.

**Nodes Involved:**  
- On form submission  
- Get a temporary file url for the upload

**Node Details:**  

- **On form submission**  
  - Type: Form Trigger  
  - Role: Entry point; triggers workflow on external form submission  
  - Config: Listens for submissions, passes form data downstream  
  - Inputs: External webhook trigger  
  - Outputs: Form data with uploaded files  
  - Failure Modes: Webhook timeout, invalid form data  

- **Get a temporary file url for the upload**  
  - Type: HTTP Request  
  - Role: Retrieves temporary accessible URL for uploaded file  
  - Config: Calls API endpoint to generate temp URL for file storage  
  - Inputs: Data from form submission node (file references)  
  - Outputs: Temporary URL for uploaded asset  
  - Failure Modes: API request failure, permission denied on file access  

---

#### 2.2 AI Copywriting & Background Image Prompt Generation

**Overview:**  
Generates marketing copy and background image prompts using OpenAI GPT-4 models orchestrated via Langchain agents, then parses structured AI outputs.

**Nodes Involved:**  
- Background Image Prompt Agent  
- Copywriting Agent  
- gpt-4.1-mini (OpenAI model for background prompt)  
- gpt-4o-mini (OpenAI model for copywriting)  
- BG Prompt Output Parser  
- Copy Output Parser  
- Merge

**Node Details:**  

- **Background Image Prompt Agent**  
  - Type: Langchain Agent  
  - Role: Generates prompts describing background images suitable for product ads  
  - Config: Uses GPT-4.1-mini as language model, receives parsed input from BG Prompt Output Parser  
  - Inputs: Temporary file URL from earlier node, merged with marketing copy output  
  - Outputs: Background image prompt data  
  - Failures: Language model API errors, parsing failures  

- **Copywriting Agent**  
  - Type: Langchain Agent  
  - Role: Creates marketing copy tailored to product and ad context  
  - Config: Uses GPT-4o-mini model, input parsed by Copy Output Parser  
  - Inputs: Temporary file URL, form data  
  - Outputs: Structured marketing copy  
  - Failures: API errors, invalid output format  

- **gpt-4.1-mini**  
  - Type: Langchain OpenAI Chat Model  
  - Role: Provides AI completions to Background Image Prompt Agent  
  - Config: GPT-4 variant with mini token limits  
  - Inputs: Prompt from BG Prompt Output Parser  
  - Outputs: Text completions  
  - Failures: Quota limits, network errors  

- **gpt-4o-mini**  
  - Type: Langchain OpenAI Chat Model  
  - Role: Provides AI completions to Copywriting Agent  
  - Config: Similar to gpt-4.1-mini but tuned for copywriting  
  - Inputs: Prompt from Copy Output Parser  
  - Outputs: Text completions  
  - Failures: Same as above  

- **BG Prompt Output Parser**  
  - Type: Langchain Structured Output Parser  
  - Role: Parses AI output into structured background prompt format  
  - Inputs: Raw text from GPT-4.1-mini  
  - Outputs: Structured prompt for image generation  
  - Failures: Parsing errors, malformed AI output  

- **Copy Output Parser**  
  - Type: Langchain Structured Output Parser  
  - Role: Parses AI output into marketing copy fields  
  - Inputs: Raw text from GPT-4o-mini  
  - Outputs: Structured marketing copy  
  - Failures: Parsing errors  

- **Merge**  
  - Type: Merge Node  
  - Role: Combines background prompt and copywriting outputs into one data stream  
  - Inputs: Outputs of Background Image Prompt Agent and Copywriting Agent  
  - Outputs: Unified data for next processing block  

---

#### 2.3 Background Image Generation & Processing via Fal.ai

**Overview:**  
Generates background images via Fal.ai based on AI prompts, polls for generation status, replaces backgrounds, and applies time-of-day relighting effects.

**Nodes Involved:**  
- Fal.ai Google Image Gen4  
- Fetch Status2  
- Wait2  
- Is Ready?2  
- Fetch Style Reference Image  
- Fal.ai Replace Background  
- Fetch Status  
- Wait  
- Is Ready?  
- Fetch Result  
- Fal.ai Time of Day Relight  
- Fetch Status1  
- Wait1  
- Is Ready?1  
- Fetch Result1

**Node Details:**  

- **Fal.ai Google Image Gen4**  
  - Type: HTTP Request  
  - Role: Sends image generation request to Fal.ai Google Imagen API (Gen4) using background prompts  
  - Inputs: Structured background prompt from Merge node  
  - Outputs: Job ID or task handle for image generation  
  - Failures: API errors, invalid prompt format  

- **Fetch Status2, Wait2, Is Ready?2**  
  - Type: HTTP Request, Wait, If nodes  
  - Role: Polls Fal.ai API for image generation completion status with wait intervals  
  - Inputs: Job ID from image generation request  
  - Outputs: Boolean ready status, triggers next step or repeat wait  
  - Failures: Timeout, lost job ID, API errors  

- **Fetch Style Reference Image**  
  - Type: HTTP Request  
  - Role: Retrieves the generated style/reference image after ready status confirmed  
  - Inputs: Job ID, ready status  
  - Outputs: Reference image URL or binary data  
  - Failures: Download errors, missing image  

- **Fal.ai Replace Background**  
  - Type: HTTP Request  
  - Role: Sends request to replace background in product images with generated background  
  - Inputs: Reference image, original product image URL  
  - Outputs: Job ID for background replacement task  
  - Failures: API errors, image incompatibility  

- **Fetch Status, Wait, Is Ready?, Fetch Result**  
  - Type: HTTP Request, Wait, If nodes  
  - Role: Polls Fal.ai for background replacement job completion and fetches results  
  - Inputs/Outputs: Job IDs and status similar to previous polling nodes  
  - Failures: Same as above  

- **Fal.ai Time of Day Relight**  
  - Type: HTTP Request  
  - Role: Applies relighting effects based on time of day to the replaced background image  
  - Inputs: Output image from replacement step  
  - Outputs: Job ID for relighting process  
  - Failures: API errors, invalid image data  

- **Fetch Status1, Wait1, Is Ready?1, Fetch Result1**  
  - Type: HTTP Request, Wait, If nodes  
  - Role: Polls and fetches results of relighting operation  
  - Inputs/Outputs: Job IDs and status  
  - Failures: Same as above  

---

#### 2.4 Image Composition with Placid.app

**Overview:**  
Uses Placid.app templating API to create final marketing images, incorporating AI-generated copy and processed background images.

**Nodes Involved:**  
- Create Image via Template with Placid.app  
- Download Image

**Node Details:**  

- **Create Image via Template with Placid.app**  
  - Type: HTTP Request  
  - Role: Calls Placid.app API to generate composite images based on templates and provided data  
  - Inputs: Marketing copy, relighted background images, template parameters  
  - Outputs: URL or job ID for generated asset  
  - Failures: API errors, template misconfiguration, data format issues  

- **Download Image**  
  - Type: HTTP Request  
  - Role: Downloads final generated image from Placid.app URL  
  - Inputs: Image URL from previous node  
  - Outputs: Binary image data for storage  
  - Failures: Network errors, invalid URL  

---

#### 2.5 Asset Download & Storage

**Overview:**  
Saves the downloaded marketing assets to Google Drive for storage and easy access.

**Nodes Involved:**  
- Save to Google Drive

**Node Details:**  

- **Save to Google Drive**  
  - Type: Google Drive Node  
  - Role: Uploads the final image files to configured Google Drive folder  
  - Config: Requires Google Drive OAuth2 credentials, target folder setup  
  - Inputs: Binary image data  
  - Outputs: Metadata of saved file (file ID, link)  
  - Failures: Authentication errors, quota exceeded, permission denied  

---

### 3. Summary Table

| Node Name                      | Node Type                           | Functional Role                                   | Input Node(s)                  | Output Node(s)                | Sticky Note                          |
|--------------------------------|-----------------------------------|-------------------------------------------------|-------------------------------|------------------------------|------------------------------------|
| On form submission             | Form Trigger                      | Entry point, triggers workflow on form submit   | -                             | Get a temporary file url for the upload |                                    |
| Get a temporary file url for the upload | HTTP Request                    | Retrieves temp URL for uploaded image            | On form submission             | Background Image Prompt Agent, Copywriting Agent |                                    |
| Background Image Prompt Agent  | Langchain Agent                   | Generates background image prompt via GPT-4.1   | Get a temporary file url for the upload | Merge                        |                                    |
| Copywriting Agent              | Langchain Agent                   | Generates marketing copy via GPT-4o-mini         | Get a temporary file url for the upload | Merge                        |                                    |
| gpt-4.1-mini                  | Langchain LM Chat OpenAI          | Provides AI completions for background prompt    | BG Prompt Output Parser        | Background Image Prompt Agent |                                    |
| gpt-4o-mini                   | Langchain LM Chat OpenAI          | Provides AI completions for copywriting          | Copy Output Parser             | Copywriting Agent             |                                    |
| BG Prompt Output Parser       | Langchain Output Parser Structured | Parses background prompt from AI text            | gpt-4.1-mini                  | Background Image Prompt Agent |                                    |
| Copy Output Parser            | Langchain Output Parser Structured | Parses marketing copy from AI text                | gpt-4o-mini                   | Copywriting Agent             |                                    |
| Merge                        | Merge Node                       | Combines background prompt and copywriting output | Background Image Prompt Agent, Copywriting Agent | Fal.ai Google Image Gen4       |                                    |
| Fal.ai Google Image Gen4      | HTTP Request                    | Requests background image generation (Fal.ai)    | Merge                         | Fetch Status2                |                                    |
| Fetch Status2                | HTTP Request                    | Checks status of Fal.ai image generation          | Fal.ai Google Image Gen4       | Is Ready?2                  |                                    |
| Wait2                        | Wait                            | Waits between status polling                      | Is Ready?2 (False path)        | Fetch Status2                |                                    |
| Is Ready?2                   | If                              | Checks if image generation is complete            | Fetch Status2                 | Fetch Style Reference Image, Wait2 |                                    |
| Fetch Style Reference Image  | HTTP Request                    | Retrieves generated style/reference image         | Is Ready?2                    | Fal.ai Replace Background    |                                    |
| Fal.ai Replace Background    | HTTP Request                    | Requests background replacement using Fal.ai      | Fetch Style Reference Image    | Fetch Status                 |                                    |
| Fetch Status                 | HTTP Request                    | Checks status of background replacement job       | Fal.ai Replace Background     | Is Ready?                   |                                    |
| Wait                        | Wait                            | Waits between status polling                       | Is Ready? (False path)         | Fetch Status                 |                                    |
| Is Ready?                   | If                              | Checks if background replacement is complete      | Fetch Status                  | Fetch Result, Wait           |                                    |
| Fetch Result                | HTTP Request                    | Fetches results of background replacement          | Is Ready?                    | Fal.ai Time of Day Relight   |                                    |
| Fal.ai Time of Day Relight  | HTTP Request                    | Applies relighting effect to image                  | Fetch Result                  | Fetch Status1                |                                    |
| Fetch Status1               | HTTP Request                    | Checks status of relighting job                      | Fal.ai Time of Day Relight    | Is Ready?1                  |                                    |
| Wait1                      | Wait                            | Waits between relighting status polling             | Is Ready?1 (False path)        | Fetch Status1               |                                    |
| Is Ready?1                 | If                              | Checks if relighting is complete                     | Fetch Status1                | Fetch Result1, Wait1          |                                    |
| Fetch Result1              | HTTP Request                    | Fetches relighted image result                        | Is Ready?1                   | Create Image via Template with Placid.app |                                    |
| Create Image via Template with Placid.app | HTTP Request                    | Creates final marketing image using Placid template | Fetch Result1                | Download Image              |                                    |
| Download Image             | HTTP Request                    | Downloads final image from Placid                     | Create Image via Template with Placid.app | Save to Google Drive          |                                    |
| Save to Google Drive       | Google Drive                    | Saves final image asset to Google Drive               | Download Image               | -                            |                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Form Trigger node**  
   - Type: Form Trigger  
   - Configure webhook to receive form submissions that include product data and image uploads.  

2. **Add an HTTP Request node named "Get a temporary file url for the upload"**  
   - Purpose: Obtain temporary accessible URL for uploaded files  
   - Connect from Form Trigger node output  
   - Configure API endpoint to generate temporary file URL for the uploaded image file  

3. **Create Langchain Agent node "Background Image Prompt Agent"**  
   - Connect input from "Get a temporary file url for the upload"  
   - Use GPT-4.1-mini as underlying model (Langchain LM Chat OpenAI node)  
   - Add a structured output parser node "BG Prompt Output Parser" to parse AI text to structured prompt  
   - Link output parser to this agent's input  

4. **Create Langchain Agent node "Copywriting Agent"**  
   - Connect input from "Get a temporary file url for the upload"  
   - Use GPT-4o-mini Langchain LM Chat OpenAI node as language model  
   - Add structured output parser node "Copy Output Parser" for parsing marketing copy  
   - Link output parser accordingly  

5. **Create a Merge node**  
   - Connect outputs from Background Image Prompt Agent and Copywriting Agent to this Merge node  
   - This node combines AI-generated background prompts and copywriting data  

6. **Create HTTP Request node "Fal.ai Google Image Gen4"**  
   - Connect input from Merge node  
   - Configure to call Fal.ai Google Imagen Gen4 API with background prompt data  

7. **Create HTTP Request node "Fetch Status2"**  
   - Connect from "Fal.ai Google Image Gen4" output  
   - Configure to poll Fal.ai for status of image generation  

8. **Create If node "Is Ready?2"**  
   - Connect input from Fetch Status2  
   - Configure condition to check if Fal.ai image generation is complete  

9. **Create Wait node "Wait2"**  
   - Connect from the False path of Is Ready?2  
   - Configure delay before re-polling status  
   - Connect Wait2 output back to Fetch Status2 for polling loop  

10. **Create HTTP Request node "Fetch Style Reference Image"**  
    - Connect from True path of Is Ready?2  
    - Configure to retrieve generated background image from Fal.ai  

11. **Create HTTP Request node "Fal.ai Replace Background"**  
    - Connect from Fetch Style Reference Image output  
    - Configure to send request to Fal.ai to replace product image background  

12. **Create HTTP Request node "Fetch Status"**  
    - Connect from Fal.ai Replace Background output  
    - Configure to poll status of background replacement job  

13. **Create If node "Is Ready?"**  
    - Connect input from Fetch Status  
    - Check if background replacement is complete  

14. **Create Wait node "Wait"**  
    - Connect from False path of Is Ready?  
    - Delay before re-polling  
    - Loop back to Fetch Status  

15. **Create HTTP Request node "Fetch Result"**  
    - Connect from True path of Is Ready?  
    - Retrieve the replaced background image result  

16. **Create HTTP Request node "Fal.ai Time of Day Relight"**  
    - Connect from Fetch Result  
    - Configure to apply relighting effect to image  

17. **Create HTTP Request node "Fetch Status1"**  
    - Connect from Fal.ai Time of Day Relight output  
    - Poll for relighting job status  

18. **Create If node "Is Ready?1"**  
    - Connect input from Fetch Status1  
    - Check if relighting is complete  

19. **Create Wait node "Wait1"**  
    - Connect from False path of Is Ready?1  
    - Delay before re-polling  
    - Loop back to Fetch Status1  

20. **Create HTTP Request node "Fetch Result1"**  
    - Connect from True path of Is Ready?1  
    - Retrieve relighted image result  

21. **Create HTTP Request node "Create Image via Template with Placid.app"**  
    - Connect from Fetch Result1  
    - Configure to call Placid.app API with relighted image and marketing copy to generate final marketing asset  

22. **Create HTTP Request node "Download Image"**  
    - Connect from Placid.app node  
    - Download final generated image  

23. **Create Google Drive node "Save to Google Drive"**  
    - Connect from Download Image node  
    - Configure with Google Drive OAuth2 credentials and target folder  
    - Upload image to Drive  

---

### 5. General Notes & Resources

| Note Content                                                                                      | Context or Link                                                                                  |
|-------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| The workflow combines OpenAI GPT-4 models with Fal.ai and Placid.app services to automate marketing asset creation. | Branding and integrations are centered on AI and templated image generation APIs.              |
| Form Trigger node requires external form setup to capture product data and image uploads.        | Ensure webhook URL is accessible and form fields match expected input in workflow.             |
| Google Drive node requires OAuth2 credentials with permissions to upload files to the target folder. | Configure credentials in n8n before running workflow.                                          |
| Langchain agents use structured output parsers to ensure AI responses are correctly formatted.   | Adjust parsers if AI model responses change or for different output schema.                    |
| Polling nodes for Fal.ai jobs include Wait delays to handle asynchronous image processing.        | Timeout or API failure in polling can cause workflow to stall; consider adding error handling. |

---

**Disclaimer:** The provided text is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.