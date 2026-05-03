Create E-commerce Promotional Carousels with Gemini 2.5 & Social Publishing

https://n8nworkflows.xyz/workflows/create-e-commerce-promotional-carousels-with-gemini-2-5---social-publishing-8002


# Create E-commerce Promotional Carousels with Gemini 2.5 & Social Publishing

---

### 1. Workflow Overview

This workflow automates the creation and social media publishing of e-commerce promotional carousels using Google Gemini 2.5 AI for image generation and OpenAI GPT-4.1 for text description generation. It targets marketers or content creators aiming to produce engaging 4-image vertical (9:16) carousels for platforms like Instagram, TikTok, Facebook, and YouTube. The workflow accepts a product photo and description, generates promotional storyboard prompts, creates multiple styled images with AI, uploads those images to an image hosting service, generates a concise advertising description, and finally publishes the post across multiple social platforms.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception & Initialization:** Receives user input (photo + product description) via a form and sets API/config variables.
- **1.2 Storyboard Generation:** Uses AI agents (Google Gemini 2.5 and Langchain tools) to create a detailed storyboard JSON defining four promotional carousel images.
- **1.3 Image Generation & Processing:** Iteratively generates AI-edited images for each carousel frame, processes outputs, and uploads images to imgbb hosting.
- **1.4 Carousel Description Generation:** Uses OpenAI GPT-4.1 to generate a brief, benefit-focused description with CTA based on the storyboard prompts.
- **1.5 Final Post Aggregation & Publishing:** Aggregates all generated data and publishes the post to multiple social media platforms with the generated images and description.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception & Initialization

- **Overview:**  
  This block collects user input via a web form and initializes workflow variables such as API keys, number of images, and image size.

- **Nodes Involved:**  
  - Photo Upload Form  
  - Set APIs Vars  
  - Merge Vars + Photo1  
  - Upload Original Image to imgbb

- **Node Details:**

  - **Photo Upload Form**  
    - Type: Form Trigger (Webhook)  
    - Role: Entry point for user to upload a product photo and description.  
    - Config: Form with two fields — single file upload (photo) and textarea (product description).  
    - Input: HTTP form submission  
    - Output: JSON with uploaded file and description  
    - Edge Cases: Missing photo or description validation; large file size; unsupported MIME types  

  - **Set APIs Vars**  
    - Type: Set  
    - Role: Defines constants and credentials for API usage (e.g., number_of_images=1, image size 1024x1024, imgbb API key placeholder).  
    - Output: Static JSON variables for later nodes  
    - Edge Cases: Missing or invalid API keys can cause failures downstream.

  - **Merge Vars + Photo1**  
    - Type: Merge (combine mode by position)  
    - Role: Combines user input from form with the API variables for unified data flow.  
    - Connections: Receives data from Photo Upload Form and Set APIs Vars

  - **Upload Original Image to imgbb**  
    - Type: HTTP Request  
    - Role: Uploads the original user photo to imgbb to get a publicly accessible URL for AI image editing.  
    - Config: POST multipart/form-data to imgbb with binary photo data and API key  
    - Output: JSON including hosted image URL  
    - Edge Cases: Upload failure, network timeout, invalid API key

#### 1.2 Storyboard Generation

- **Overview:**  
  Generates a detailed JSON storyboard for 4 promotional carousel images based on the product description, guiding subsequent AI image generation.

- **Nodes Involved:**  
  - Google Gemini Chat Model  
  - Structured Output Parser2  
  - Storyboard Agent  
  - Set Storyboard Vars  
  - Think2 (AI Tool interface)

- **Node Details:**

  - **Google Gemini Chat Model**  
    - Type: Langchain Google Gemini Chat LLM node  
    - Role: Provides the language model interface for generating prompts.  
    - Config: Uses "models/gemini-2.5-pro" with Google Palm API credentials.  
    - Input: Product description from form  
    - Output: Raw AI chatbot response  
    - Edge Cases: API quota, auth errors, response latency  

  - **Think2**  
    - Type: Langchain tool (think node)  
    - Role: Intermediate AI processing step, connected as a tool to Storyboard Agent  
    - Functions as a processing helper node, no direct config

  - **Storyboard Agent**  
    - Type: Langchain Agent node  
    - Role: Core creative agent that generates a compact JSON containing 4 image prompts with descriptions, environment, and sound for a promotional carousel.  
    - Config: Input prompt instructs to create 4 independent vertical 9:16 images featuring real people using the product, with promotional text and CTAs, in authentic environments. Output is strict JSON with keys: title, prompt1-4, i2v_prompt1-4, environment, sound.  
    - Input: Product description and AI model output  
    - Output: Parsed JSON with storyboard details  
    - Edge Cases: JSON parsing errors, incomplete or invalid JSON output  

  - **Structured Output Parser2**  
    - Type: Langchain Structured Output Parser  
    - Role: Validates and parses the AI output ensuring it fits the predefined JSON schema (example provided).  
    - Input: Raw AI output from Storyboard Agent  
    - Output: Clean structured JSON for downstream use

  - **Set Storyboard Vars**  
    - Type: Set  
    - Role: Assigns individual storyboard keys (title, prompts, environment, sound) into workflow variables for easier reference downstream.  
    - Input: Parsed storyboard JSON  
    - Output: Individual JSON fields as separate variables

#### 1.3 Image Generation & Processing

- **Overview:**  
  Generates AI-edited images for each of the 4 storyboard prompts using Gemini 2.5 Flash image editing API, processes output arrays, renames and uploads resulting images back to imgbb.

- **Nodes Involved:**  
  - Gemini 2.5 Flash - Generate Image 2, 3, 4, 5  
  - Separate Image Outputs 2, 3, 4, 5  
  - Rename to photo 2, 3, 4, 5  
  - Upload Image to imgbb 2, 3, 4, 5  
  - HTTP Request, HTTP Request1, HTTP Request2, HTTP Request3

- **Node Details:**

  - **Gemini 2.5 Flash - Generate Image (2 to 5)**  
    - Type: HTTP Request  
    - Role: Calls fal.ai Gemini 2.5 Flash image editing API to generate images based on storyboard prompts and original uploaded photo URL.  
    - Config: POST JSON body includes prompt (from Set Storyboard Vars), original image URL, and number of images requested (usually 1). Uses HTTP header auth with fal.ai credentials.  
    - Output: JSON containing array of generated images  
    - Edge Cases: API auth failure, rate limiting, prompt errors, network issues  

  - **Separate Image Outputs (2 to 5)**  
    - Type: SplitOut  
    - Role: Splits the array of generated images into individual items for further processing.  
    - Input: JSON array from Gemini nodes  
    - Output: Single image items  

  - **Rename to photo (2 to 5)**  
    - Type: Code node  
    - Role: Maps split image items to a JSON structure with URL and binary data under keys photo2 to photo5, preparing for upload.  
    - Output: JSON with image URL and binary photo data keyed respectively  

  - **Upload Image to imgbb (2 to 5)**  
    - Type: HTTP Request  
    - Role: Uploads each generated image binary data to imgbb to obtain a public URL for social media posting.  
    - Config: Multipart form-data POST with image binary and imgbb API key. Retries enabled on failure.  
    - Output: JSON with hosted image URL  
    - Edge Cases: Upload failure, API key invalid, network errors  

  - **HTTP Request (0 to 3)**  
    - Type: HTTP Request  
    - Role: Fetches the URLs of the separated images from previous steps to confirm accessibility or trigger further processing.  
    - Config: Simple GET requests using URLs from separate outputs.  
    - Usage: Helps validate or cache images before upload.

#### 1.4 Carousel Description Generation

- **Overview:**  
  Creates a concise, benefit-oriented advertising description with a strong call-to-action for the carousel using OpenAI GPT-4.1.

- **Nodes Involved:**  
  - Aggregate  
  - Merge  
  - Generate Carousel Description

- **Node Details:**

  - **Merge**  
    - Type: Merge  
    - Role: Combines uploaded image URLs from all four generated images into a single data stream for downstream processing.  
    - Config: Number inputs = 4 to combine all image upload outputs.

  - **Aggregate**  
    - Type: Aggregate  
    - Role: Aggregates all incoming item data into one consolidated item for passing full data context to the description generation node.  

  - **Generate Carousel Description**  
    - Type: Langchain OpenAI node  
    - Role: Sends prompts based on four storyboard prompts to OpenAI GPT-4.1 to generate a short, engaging description for social media posts.  
    - Config: System prompt sets role as expert Instagram/TikTok description creator, user prompt includes the 4 storyboard prompts with instructions to generate ≤150 characters, benefit-focused with CTA, language matching input.  
    - Output: Single string description  
    - Edge Cases: API rate limits, inappropriate outputs, language mismatch

#### 1.5 Final Post Aggregation & Publishing

- **Overview:**  
  Aggregates all generated media and text, then publishes the post with images and description to multiple social media platforms via Upload Post node.

- **Nodes Involved:**  
  - Upload Post

- **Node Details:**

  - **Upload Post**  
    - Type: Upload Post (custom n8n node)  
    - Role: Publishes the promotional carousel post to platforms: Instagram, TikTok, YouTube, Facebook.  
    - Config: Uses user credentials, includes the generated description as title/message, and uploads the 4 imgbb-hosted images as photos. Facebook page ID configured as parameter.  
    - Input: Aggregated description and image URLs  
    - Output: Post confirmation or error  
    - Edge Cases: Platform API quota, auth token expiration, image format issues, network failure

---

### 3. Summary Table

| Node Name                      | Node Type                            | Functional Role                         | Input Node(s)               | Output Node(s)                            | Sticky Note                                           |
|-------------------------------|------------------------------------|---------------------------------------|-----------------------------|------------------------------------------|-------------------------------------------------------|
| Photo Upload Form              | Form Trigger                       | Collects photo and product description| -                           | Merge Vars + Photo1, Storyboard Agent    | Upload photo and description                           |
| Set APIs Vars                 | Set                                | Defines API keys and config variables | Set Storyboard Vars (downstream) | Merge Vars + Photo1                    |                                                       |
| Merge Vars + Photo1           | Merge (combine)                    | Combines user input with API vars     | Photo Upload Form, Set APIs Vars | Upload Original Image to imgbb         |                                                       |
| Upload Original Image to imgbb| HTTP Request                      | Uploads original photo to imgbb       | Merge Vars + Photo1           | Gemini 2.5 Flash - Generate Image 2-5    |                                                       |
| Google Gemini Chat Model      | Langchain Gemini 2.5 Chat LLM      | AI language model for storyboard      | -                           | Storyboard Agent                         |                                                       |
| Think2                       | Langchain Tool (Think)             | AI processing helper                  | -                           | Storyboard Agent                         |                                                       |
| Storyboard Agent             | Langchain Agent                    | Generates 4-image promotional storyboard JSON | Photo Upload Form, Google Gemini Chat Model, Think2 | Set Storyboard Vars                 |                                                       |
| Structured Output Parser2     | Langchain Output Parser            | Validates AI JSON output               | Storyboard Agent            | Storyboard Agent (parser output)         |                                                       |
| Set Storyboard Vars           | Set                                | Assigns storyboard JSON keys to variables | Storyboard Agent            | Set APIs Vars                            |                                                       |
| Gemini 2.5 Flash - Generate Image 2 | HTTP Request                | Generates edited image for prompt1    | Upload Original Image to imgbb | Separate Image Outputs 2                 | Uses Gemini 2.5 Flash API with fal.ai credentials     |
| Separate Image Outputs 2      | SplitOut                          | Splits image array to single images   | Gemini 2.5 Flash - Generate Image 2 | HTTP Request                            |                                                       |
| Rename to photo 2             | Code                              | Prepares image binary for upload      | HTTP Request                 | Upload Image to imgbb 2                   |                                                       |
| Upload Image to imgbb 2       | HTTP Request                      | Uploads AI-generated image to imgbb   | Rename to photo 2            | Merge                                   |                                                       |
| Gemini 2.5 Flash - Generate Image 3 | HTTP Request                | Generates edited image for prompt2    | Upload Original Image to imgbb | Separate Image Outputs 3                 |                                                       |
| Separate Image Outputs 3      | SplitOut                          | Splits image array to single images   | Gemini 2.5 Flash - Generate Image 3 | HTTP Request1                           |                                                       |
| Rename to photo 3             | Code                              | Prepares image binary for upload      | HTTP Request1                | Upload Image to imgbb 3                   |                                                       |
| Upload Image to imgbb 3       | HTTP Request                      | Uploads AI-generated image to imgbb   | Rename to photo 3            | Merge                                   |                                                       |
| Gemini 2.5 Flash - Generate Image 4 | HTTP Request                | Generates edited image for prompt3    | Upload Original Image to imgbb | Separate Image Outputs 4                 |                                                       |
| Separate Image Outputs 4      | SplitOut                          | Splits image array to single images   | Gemini 2.5 Flash - Generate Image 4 | HTTP Request2                           |                                                       |
| Rename to photo 4             | Code                              | Prepares image binary for upload      | HTTP Request2                | Upload Image to imgbb 4                   |                                                       |
| Upload Image to imgbb 4       | HTTP Request                      | Uploads AI-generated image to imgbb   | Rename to photo 4            | Merge                                   |                                                       |
| Gemini 2.5 Flash - Generate Image 5 | HTTP Request                | Generates edited image for prompt4    | Upload Original Image to imgbb | Separate Image Outputs 5                 |                                                       |
| Separate Image Outputs 5      | SplitOut                          | Splits image array to single images   | Gemini 2.5 Flash - Generate Image 5 | HTTP Request3                           |                                                       |
| Rename to photo 5             | Code                              | Prepares image binary for upload      | HTTP Request3                | Upload Image to imgbb 5                   |                                                       |
| Upload Image to imgbb 5       | HTTP Request                      | Uploads AI-generated image to imgbb   | Rename to photo 5            | Merge                                   |                                                       |
| Merge                        | Merge                             | Combines all uploaded image URLs      | Upload Image to imgbb 2-5    | Aggregate                                |                                                       |
| Aggregate                   | Aggregate                         | Aggregates all data for description    | Merge                       | Generate Carousel Description            |                                                       |
| Generate Carousel Description | Langchain OpenAI                  | Creates short social media description| Aggregate                   | Upload Post                              | Uses GPT-4.1, outputs ≤150 char benefit-focused CTA   |
| Upload Post                  | Upload Post (custom node)          | Publishes carousel and description    | Generate Carousel Description | -                                        | Posts to Instagram, TikTok, YouTube, Facebook         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Form Trigger node named "Photo Upload Form":**  
   - Path: `/generate-ad`  
   - Fields:  
     - File upload (single) labeled "photo" (required)  
     - Textarea labeled "Product description" (required)  

2. **Create a Set node "Set APIs Vars":**  
   - Assign static values:  
     - number_of_images = 1  
     - size_of_image = "1024x1024"  
     - openai_image_model = "gemini-25-flash-image"  
     - format_image = "webp"  
     - imgbb_api_key = [Your imgbb API key]  

3. **Create a Merge node "Merge Vars + Photo1":**  
   - Mode: Combine  
   - Combine by position  
   - Connect "Photo Upload Form" and "Set APIs Vars" to this node  

4. **Create HTTP Request node "Upload Original Image to imgbb":**  
   - Method: POST  
   - URL: `https://api.imgbb.com/1/upload`  
   - Body Type: multipart/form-data  
   - Parameters:  
     - image: mapped to binary data from form photo upload  
     - key: `={{ $('Set APIs Vars').item.json.imgbb_api_key }}`  
   - Retry on fail: enabled  

5. **Set up Google Gemini Chat Model node (Langchain LLM):**  
   - Model: `models/gemini-2.5-pro`  
   - Credentials: Google Palm API key  

6. **Add a "Think2" node (Langchain Tool) for AI processing.**

7. **Create "Storyboard Agent" (Langchain Agent):**  
   - Input: product description from form  
   - Prompt: As per node, instructing to create 4-image carousel JSON with people, promotional text, environment, CTAs, and strict JSON output.  
   - Attach "Think2" as AI tool and "Google Gemini Chat Model" as language model.  
   - Attach structured output parser "Structured Output Parser2" for JSON validation.  
   - Set system message and prompt as specified in original config.  

8. **Create "Set Storyboard Vars" (Set node):**  
   - Assign all keys from storyboard JSON output (title, prompt1-4, i2v_prompt1-4, environment, sound) to individual variables.  

9. **Create 4 HTTP Request nodes for image generation ("Gemini 2.5 Flash - Generate Image 2", 3, 4, 5):**  
   - Method: POST  
   - URL: `https://fal.run/fal-ai/gemini-25-flash-image/edit`  
   - Body JSON: { "prompt": promptN, "image_urls": [original image URL], "num_images": 1 }  
   - Authentication: HTTP Header Auth with fal.ai credentials  
   - Use prompts from "Set Storyboard Vars" (prompt1 to prompt4)  

10. **Create 4 SplitOut nodes ("Separate Image Outputs 2" to 5):**  
    - Field to split: "images"  

11. **Create 4 Code nodes ("Rename to photo 2" to 5):**  
    - JavaScript to map image binary data and URL to JSON with keys photoN  

12. **Create 4 HTTP Request nodes ("Upload Image to imgbb 2" to 5):**  
    - Method: POST  
    - URL: `https://api.imgbb.com/1/upload`  
    - Body: multipart/form-data with binary photoN and imgbb key  
    - Retry enabled  

13. **Create a Merge node "Merge":**  
    - Number Inputs: 4  
    - Connect all 4 upload image nodes to this merge node  

14. **Create an Aggregate node "Aggregate":**  
    - Aggregate all item data into one  

15. **Create OpenAI node "Generate Carousel Description":**  
    - Model: GPT-4.1  
    - System message: Expert Instagram/TikTok carousel description creator  
    - User message: Provide storyboard prompts 1-4, ask for ≤150 chars benefit-oriented description with CTA  
    - Credentials: OpenAI API Key  

16. **Create "Upload Post" node:**  
    - User: set appropriately  
    - Title: bind to generated description from "Generate Carousel Description"  
    - Photos: comma-separated URLs from uploaded images 2-5  
    - Platforms: Instagram, TikTok, YouTube, Facebook  
    - Facebook Page ID: your page ID  
    - Credentials: Upload Post API credentials  

17. **Connect nodes as per the original flow:**  
    - Form trigger → Merge Vars + Photo1 → Upload Original Image to imgbb → Gemini 2.5 Flash Generate Image nodes (2-5) → Separate Image Outputs → Rename → Upload Image to imgbb → Merge → Aggregate → Generate Carousel Description → Upload Post  
    - Form trigger → Storyboard Agent (with Gemini Chat and Think2) → Structured Output Parser2 → Set Storyboard Vars → Set APIs Vars → Merge Vars + Photo1  

18. **Set all credentials:**  
    - Google Palm API for Gemini Chat  
    - fal.ai credentials for image generation  
    - imgbb API key for image hosting  
    - OpenAI API for description generation  
    - Upload Post API credentials for publishing  

---

### 5. General Notes & Resources

| Note Content                                                                                  | Context or Link                                               |
|-----------------------------------------------------------------------------------------------|---------------------------------------------------------------|
| The workflow uses fal.ai Gemini 2.5 Flash image editing API for rapid AI-based visual editing.| fal.ai API: https://fal.run/fal-ai/gemini-25-flash-image/edit  |
| imgbb is used as a public image hosting service to generate URLs for social media posts.      | imgbb API Docs: https://api.imgbb.com/                        |
| OpenAI GPT-4.1 is leveraged for concise social media ad text generation with strong CTAs.     | OpenAI API Docs: https://platform.openai.com/docs             |
| Upload Post node supports multi-platform social publishing, including Instagram, TikTok, etc. | Custom n8n node with platform-specific API integration          |
| The storyboard JSON schema enforces strict formatting for AI output to avoid parsing errors.  | Example schema provided in Structured Output Parser2 node     |
| User inputs must include both photo and product description for the workflow to function fully.| Form validation recommended                                    |

---

**Disclaimer:** The text provided is derived exclusively from an automated workflow created with n8n, a tool for integration and automation. This processing strictly respects applicable content policies and contains no illegal, offensive, or protected elements. All processed data is legal and public.