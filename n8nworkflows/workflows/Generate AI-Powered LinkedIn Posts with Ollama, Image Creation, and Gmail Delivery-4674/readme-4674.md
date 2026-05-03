Generate AI-Powered LinkedIn Posts with Ollama, Image Creation, and Gmail Delivery

https://n8nworkflows.xyz/workflows/generate-ai-powered-linkedin-posts-with-ollama--image-creation--and-gmail-delivery-4674


# Generate AI-Powered LinkedIn Posts with Ollama, Image Creation, and Gmail Delivery

### 1. Workflow Overview

This workflow automates the generation of AI-powered LinkedIn posts enriched with custom images and delivers the final content via Gmail email. It is designed to streamline content creation for social media marketing or personal branding by leveraging AI language models and image generation APIs.

**Target Use Cases:**  
- Automatic creation of LinkedIn posts based on user input from a form  
- AI-driven generation of creative image prompts and corresponding images  
- Automated email delivery of the generated posts and images  
- Optional manual triggering for testing the image generation process  

**Logical Blocks:**

- **1.1 Input Reception:** Triggers the workflow on form submission or manual trigger  
- **1.2 AI Text Generation:** Generates LinkedIn post content using a LangChain AI agent powered by the Ollama Chat Model and external HTTP tools  
- **1.3 AI Image Prompt Generation:** Creates image prompts from the LinkedIn post using a second LangChain AI agent with Ollama Chat Model  
- **1.4 Image Generation and Processing:** Uses HTTP requests to generate images based on the AI prompts, converts images to files, and downloads them  
- **1.5 Delivery:** Sends emails via Gmail with the generated content and posts the content to LinkedIn  

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  Initiates the workflow either when a form is submitted or when manually triggered for testing. Provides the initial input data for AI processing.

- **Nodes Involved:**  
  - On form submission  
  - When clicking ‘Test workflow’

- **Node Details:**  

  - **On form submission**  
    - *Type & Role:* Form Trigger node that listens for HTTP POST with form data  
    - *Configuration:* Configured with a webhook ID to receive form submissions  
    - *Expressions/Variables:* Captures form fields as JSON input for downstream nodes  
    - *Connections:* Outputs to "LinkedIn Post Agent"  
    - *Edge Cases:* Failures if webhook is not reachable or form data is malformed  

  - **When clicking ‘Test workflow’**  
    - *Type & Role:* Manual Trigger for testing workflow operation  
    - *Configuration:* Default manual trigger; used to test image generation  
    - *Connections:* Outputs to "Generate Image1"  
    - *Edge Cases:* None; purely manual  

#### 1.2 AI Text Generation

- **Overview:**  
  Generates the main LinkedIn post content using AI language models, interfacing with both LangChain agents and Ollama Chat Model for natural language generation.

- **Nodes Involved:**  
  - LinkedIn Post Agent  
  - Ollama Chat Model  
  - Tavily (HTTP tool node)

- **Node Details:**  

  - **LinkedIn Post Agent**  
    - *Type & Role:* LangChain Agent node that orchestrates AI language model calls  
    - *Configuration:* Uses Ollama Chat Model as language model; integrates with Tavily HTTP tool for external API calls  
    - *Expressions/Variables:* Uses input from form submission or prior nodes as prompt  
    - *Connections:* Input from "On form submission", output to "Image Prompt Agent"  
    - *Edge Cases:* AI model errors, API rate limits, invalid prompts  

  - **Ollama Chat Model**  
    - *Type & Role:* AI language model node using Ollama for chat completions  
    - *Configuration:* Configured with Ollama credentials and prompt templates  
    - *Connections:* Feeds into "LinkedIn Post Agent" as language model backend  
    - *Edge Cases:* Authentication errors, network timeouts  

  - **Tavily**  
    - *Type & Role:* HTTP Request tool integrated as an AI tool node for LinkedIn Post Agent  
    - *Configuration:* External HTTP API calls enhancing AI capabilities (details abstracted)  
    - *Connections:* Used as an AI tool by "LinkedIn Post Agent"  
    - *Edge Cases:* HTTP errors, API downtime  

#### 1.3 AI Image Prompt Generation

- **Overview:**  
  Transforms the LinkedIn post text into an image prompt using a separate LangChain Agent with Ollama Chat Model, preparing for image generation.

- **Nodes Involved:**  
  - Image Prompt Agent  
  - Ollama Chat Model1  

- **Node Details:**  

  - **Image Prompt Agent**  
    - *Type & Role:* LangChain Agent node generating image prompts from text input  
    - *Configuration:* Uses Ollama Chat Model1 as language model backend  
    - *Connections:* Input from "LinkedIn Post Agent", output to "Generate Image2"  
    - *Edge Cases:* AI model misinterpretation, invalid prompt generation  

  - **Ollama Chat Model1**  
    - *Type & Role:* Ollama Chat Model node dedicated to image prompt creation  
    - *Configuration:* Similar to Ollama Chat Model but tailored for image prompt generation  
    - *Connections:* Feeds into "Image Prompt Agent" as language model  
    - *Edge Cases:* Same as Ollama Chat Model  

#### 1.4 Image Generation and Processing

- **Overview:**  
  Generates images from the AI prompts, converts and downloads them, preparing files for email attachment and LinkedIn posting.

- **Nodes Involved:**  
  - Generate Image1  
  - Convert to File  
  - Generate Image2  
  - Download Image  

- **Node Details:**  

  - **Generate Image1**  
    - *Type & Role:* HTTP Request node to generate image (used in manual test flow)  
    - *Configuration:* Calls image generation API with prompt data  
    - *Connections:* Output to "Convert to File"  
    - *Edge Cases:* API failures, invalid responses  

  - **Convert to File**  
    - *Type & Role:* Converts raw image data to file object compatible with n8n  
    - *Configuration:* Default file conversion settings  
    - *Connections:* Input from "Generate Image1"  
    - *Edge Cases:* Data format errors  

  - **Generate Image2**  
    - *Type & Role:* HTTP Request node generating image from AI prompt during form submission flow  
    - *Configuration:* Calls image generation API with prompt from "Image Prompt Agent"  
    - *Connections:* Output to "Download Image"  
    - *Edge Cases:* Timeout, API errors  

  - **Download Image**  
    - *Type & Role:* HTTP Request node downloading the generated image file for further use  
    - *Configuration:* Uses URL from "Generate Image2" response to fetch image  
    - *Connections:* Output to "Send Email"  
    - *Edge Cases:* Download failures, broken links  

#### 1.5 Delivery

- **Overview:**  
  Sends the generated LinkedIn post and image via Gmail email and posts the content to LinkedIn.

- **Nodes Involved:**  
  - Send Email  
  - LinkedIn  

- **Node Details:**  

  - **Send Email**  
    - *Type & Role:* Email Send node configured for Gmail SMTP or OAuth2  
    - *Configuration:* Email composed with LinkedIn post text and attached generated image file  
    - *Connections:* Input from "Download Image", output to "LinkedIn"  
    - *Edge Cases:* Authentication errors, email delivery failures  

  - **LinkedIn**  
    - *Type & Role:* LinkedIn node posting content to LinkedIn platform  
    - *Configuration:* Uses OAuth2 credentials for authorization  
    - *Connections:* Input from "Send Email"  
    - *Edge Cases:* API rate limiting, authorization errors  


---

### 3. Summary Table

| Node Name              | Node Type                           | Functional Role                        | Input Node(s)           | Output Node(s)          | Sticky Note                   |
|------------------------|-----------------------------------|-------------------------------------|------------------------|-------------------------|------------------------------|
| On form submission     | Form Trigger                      | Receive user form input               | —                      | LinkedIn Post Agent      |                              |
| When clicking ‘Test workflow’ | Manual Trigger               | Manual start for testing              | —                      | Generate Image1          |                              |
| LinkedIn Post Agent    | LangChain Agent                   | Generate LinkedIn post text           | On form submission      | Image Prompt Agent       |                              |
| Ollama Chat Model      | Ollama Chat Model                 | AI language model for post generation | —                      | LinkedIn Post Agent      |                              |
| Tavily                 | LangChain HTTP Tool              | External AI API calls                  | —                      | LinkedIn Post Agent (tool) |                              |
| Image Prompt Agent     | LangChain Agent                   | Generate image prompt from post text  | LinkedIn Post Agent     | Generate Image2          |                              |
| Ollama Chat Model1     | Ollama Chat Model                 | AI language model for image prompt    | —                      | Image Prompt Agent       |                              |
| Generate Image1        | HTTP Request                     | Generate image (manual test)           | When clicking ‘Test workflow’ | Convert to File      |                              |
| Convert to File        | Convert to File                   | Convert image data to file             | Generate Image1         | —                       |                              |
| Generate Image2        | HTTP Request                     | Generate image from AI prompt          | Image Prompt Agent      | Download Image           |                              |
| Download Image         | HTTP Request                     | Download generated image               | Generate Image2         | Send Email               |                              |
| Send Email             | Email Send                       | Send email with post and image         | Download Image          | LinkedIn                 |                              |
| LinkedIn               | LinkedIn API                    | Post content to LinkedIn               | Send Email              | —                       |                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Nodes:**  
   - Add **Form Trigger** node, configure webhook with unique ID to receive form submissions.  
   - Add **Manual Trigger** node for testing purposes.

2. **Add LinkedIn Post AI Generation:**  
   - Add **LangChain Agent** node named “LinkedIn Post Agent.”  
   - Configure it to use **Ollama Chat Model** as the language model backend node.  
   - Add **Ollama Chat Model** node: configure credentials and prompt templates for post text generation.  
   - Add **Tavily** node configured as an HTTP tool to support LinkedIn Post Agent for external API calls (set base URL and authentication as required).  
   - Connect **Form Trigger** output to **LinkedIn Post Agent** input.  
   - Connect **Ollama Chat Model** to **LinkedIn Post Agent** as language model backend.  
   - Configure **Tavily** as AI tool for LinkedIn Post Agent.

3. **Add Image Prompt Generation:**  
   - Add **LangChain Agent** node named “Image Prompt Agent.”  
   - Add **Ollama Chat Model1** node for image prompt generation (separate Ollama node with credentials).  
   - Connect **LinkedIn Post Agent** output to **Image Prompt Agent** input.  
   - Connect **Ollama Chat Model1** as language model backend to **Image Prompt Agent**.

4. **Add Image Generation and Processing:**  
   - Add **HTTP Request** node named “Generate Image2” to call image generation API using prompt from **Image Prompt Agent**.  
   - Add **HTTP Request** node named “Download Image” to download the generated image file using URL returned by “Generate Image2.”  
   - Connect **Image Prompt Agent** output to **Generate Image2** input, then **Generate Image2** output to **Download Image** input.

5. **Add Email Delivery and LinkedIn Posting:**  
   - Add **Email Send** node named “Send Email.” Configure with Gmail OAuth2 or SMTP credentials.  
   - Set email body to include LinkedIn post text and attach file from **Download Image**.  
   - Add **LinkedIn** node; configure OAuth2 credentials for posting.  
   - Connect **Download Image** output to **Send Email** input.  
   - Connect **Send Email** output to **LinkedIn** input.

6. **Add Manual Test Image Generation Flow (Optional):**  
   - Add **HTTP Request** node named “Generate Image1” for test image generation.  
   - Add **Convert to File** node connected to “Generate Image1” to convert raw data.  
   - Connect **Manual Trigger** output to “Generate Image1” input.

7. **Final Checks:**  
   - Set all API credentials (Ollama, Gmail, LinkedIn, image generation API).  
   - Verify webhook URLs for form submission and email sending.  
   - Test manual trigger to ensure image generation works independently.  
   - Test full form submission flow end-to-end.

---

### 5. General Notes & Resources

| Note Content                                                                                          | Context or Link                                 |
|-----------------------------------------------------------------------------------------------------|------------------------------------------------|
| The workflow leverages Ollama for AI chat completions, allowing flexible prompt engineering.         | https://ollama.com/                             |
| Ensure OAuth2 credentials for LinkedIn and Gmail are set up with correct scopes for posting and sending. | LinkedIn Developer Portal and Google Cloud Console |
| Image generation API endpoints and parameters must be configured to match your chosen provider's spec.  | Example providers: DALL·E, Stable Diffusion APIs |
| For troubleshooting AI model errors, inspect prompt variables and API usage limits carefully.        | n8n Documentation on LangChain Agents          |

---

**Disclaimer:** The provided text is extracted exclusively from an automated workflow created using n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All processed data is legal and public.