Generate Interior Moodboards with Claude/Gemini Agents, Hugging Face Image Generation, and PDF Export

https://n8nworkflows.xyz/workflows/generate-interior-moodboards-with-claude-gemini-agents--hugging-face-image-generation--and-pdf-export-10880


# Generate Interior Moodboards with Claude/Gemini Agents, Hugging Face Image Generation, and PDF Export

### 1. Workflow Overview

This n8n workflow automates the creation of interior design moodboards by integrating AI language models (Google Gemini and OpenRouter’s Claude), image generation via Hugging Face, and automated PDF report generation with email delivery. It targets interior designers or clients seeking detailed, AI-generated moodboards representing design concepts visually and in print-ready formats.

The workflow is logically grouped into three main functional blocks:

- **1.1 Input Reception & Preprocessing:** Captures moodboard requests via a web form, extracts user email parts, and prepares personalized storage folders in Nextcloud.

- **1.2 AI Conceptualization & Image Prompt Generation:** Uses AI agents to analyze input briefs, produce detailed image prompts in JSON, and parse/split these prompts for image generation.

- **1.3 Image Generation, Storage & Final Report Assembly:** Iterates over prompts to generate images via Hugging Face, uploads images to Nextcloud, generates public URLs, aggregates URLs, constructs a two-page HTML moodboard, converts it to PDF, and emails the final deliverable to the user.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Preprocessing

**Overview:**  
This block collects user input from an n8n form, extracts the email local part for folder naming, and creates a dedicated Nextcloud folder to store generated images.

**Nodes Involved:**  
- Moodbaord Form (formTrigger)  
- Email extractor (Code)  
- Create a folder (Nextcloud)  
- Sticky Note (explanations)

**Node Details:**

- **Moodbaord Form**  
  - Type: Form Trigger  
  - Role: Captures user input including moodboard title, description, and email.  
  - Configuration: Form fields include a required text title, a detailed description textarea, and a required email field. Timestamp and test/live mode also recorded.  
  - Input: User web submission  
  - Output: JSON with user inputs  
  - Edge Cases: Missing required fields, malformed email input, form submission failures.

- **Email extractor**  
  - Type: Code (JavaScript)  
  - Role: Extracts the local part of the user's email address before the '@' symbol, stripping optional '+' tags.  
  - Key expression: Retrieves email from form JSON; splits on '@' and '+' to clean local part.  
  - Input: Output of Moodbaord Form  
  - Output: JSON object with `localPart` (string) for folder naming  
  - Edge Cases: Empty email, unexpected format, missing input key.

- **Create a folder**  
  - Type: Nextcloud API node  
  - Role: Creates a folder in Nextcloud at path `/moodboard/<localPart>`, personalized per user.  
  - Configuration: Uses extracted local part in folder path  
  - Credentials: Nextcloud account with folder creation permissions  
  - Input: Local part from Email extractor  
  - Output: Folder creation confirmation or error (continues on error)  
  - Edge Cases: Folder already exists, Nextcloud auth failure, network errors.

- **Sticky Notes in this block** explain the form's purpose and the email extraction logic for clarity.

---

#### 2.2 AI Conceptualization & Image Prompt Generation

**Overview:**  
This block sends the user’s design brief to a custom AI agent that conceptualizes an interior design moodboard, generating 12 detailed image prompts in JSON format. The output is parsed and split into individual image prompts for further processing.

**Nodes Involved:**  
- Think2 (Langchain toolThink)  
- OpenRouter Chat Model (Claude)  
- Simple Memory (Langchain memoryBufferWindow)  
- Structured Output Parser2  
- Conceptualization Agent (Langchain Agent)  
- Concept Splitter (splitOut node)  
- Sticky Notes (explanations)

**Node Details:**

- **Think2**  
  - Type: Langchain toolThink (internal AI processing step)  
  - Role: Supports internal conceptual analysis before prompt generation.  
  - Input: Passes data to Conceptualization Agent.

- **OpenRouter Chat Model**  
  - Type: Langchain AI Language Model node (Claude)  
  - Role: Provides conversational AI processing for the agent.  
  - Credentials: OpenRouter API key  
  - Edge Cases: API quota, latency, or auth failures.

- **Simple Memory**  
  - Type: Langchain memory buffer  
  - Role: Maintains context/session by using the moodboard title as a session key.  
  - Input: Form title for session isolation.

- **Structured Output Parser2**  
  - Type: Langchain Structured Output Parser  
  - Role: Parses the AI JSON output containing 12 image prompts, each 300-500 words describing visual details.  
  - Configuration: Uses a JSON schema example defining prompts `Image_prompt_1` to `Image_prompt_12_Complete_3D_View`.  
  - Input: Raw AI response  
  - Output: Parsed JSON object with all prompts

- **Conceptualization Agent**  
  - Type: Langchain Agent  
  - Role: Main AI agent that receives the title and description, runs the conceptual analysis, and generates the structured JSON of image prompts.  
  - System Message: Detailed prompt instructing the agent to generate 12 highly detailed image prompts with specific instructions for each, including a comprehensive 3D rendered view as the 12th prompt.  
  - Inputs: Title and description from form; uses memory and AI models.  
  - Outputs: JSON with prompts  
  - Edge Cases: AI response formatting errors, API timeouts, malformed JSON output.

- **Concept Splitter**  
  - Type: splitOut node  
  - Role: Splits the JSON object containing 12 prompts into 12 individual items for parallel or sequential processing.  
  - Input: JSON from Conceptualization Agent  
  - Output: 12 separate JSON items, each with one prompt.

- **Sticky Notes** explain the conceptualization agent’s role and the splitting of prompts for parallel processing.

---

#### 2.3 Image Generation, Storage & Final Report Assembly

**Overview:**  
This block loops through each of the 12 image prompts to generate images using the Hugging Face API, uploads them to Nextcloud, generates public sharing URLs, aggregates all URLs, generates a two-page HTML moodboard, converts it into PDF, and sends the final PDF by email.

**Nodes Involved:**  
- Loop Over Items (splitInBatches)  
- Image Generator (HTTP Request - Hugging Face)  
- Upload Image (Nextcloud)  
- Share a file (Nextcloud)  
- set image title and url (Set node)  
- URL Aggregate (Aggregate node)  
- Clean URLs (Code node)  
- Moodboard Generator Agent (Langchain Agent)  
- Binary Converter (Code node)  
- PDF creator (HTTP Request - Gotenberg)  
- Send PDF (Gmail)  
- Sticky Notes (explanations)

**Node Details:**

- **Loop Over Items**  
  - Type: splitInBatches  
  - Role: Processes each image prompt sequentially to control API requests and resource usage.  
  - Input: 12 individual prompts from Concept Splitter  
  - Output: Sends each prompt to image generation and subsequent nodes.

- **Image Generator**  
  - Type: HTTP Request  
  - Role: Sends detailed image prompt text to Hugging Face’s image generation API (model "black-forest-labs/FLUX.1-schnell"), requesting JPEG images.  
  - Authentication: HTTP header with Hugging Face token  
  - Input: Prompt text from loop  
  - Output: Binary image data  
  - Edge Cases: API quota limits, timeouts, malformed prompt issues.

- **Upload Image**  
  - Type: Nextcloud API node  
  - Role: Uploads the generated image to the user’s personalized Nextcloud folder with the prompt title as filename.  
  - Binary Upload: Enabled  
  - Input: Image binary from Image Generator  
  - Output: Confirmation and file path  
  - Edge Cases: Upload failures, auth errors.

- **Share a file**  
  - Type: Nextcloud API node  
  - Role: Creates a public share link for each uploaded image with permissions allowing external access.  
  - Input: Uploaded file path  
  - Output: Public URL for image  
  - Edge Cases: Share creation failures, permission issues.

- **set image title and url**  
  - Type: Set node  
  - Role: Formats and attaches the image title and public URL (with `/preview` appended) for aggregation.  
  - Input: Data from Share a file node.

- **URL Aggregate**  
  - Type: Aggregate node  
  - Role: Collects all image URLs into a single list in preparation for final moodboard generation.  
  - Input: Multiple URL items from loop output.

- **Clean URLs**  
  - Type: Code node  
  - Role: Extracts all URLs from aggregated data into a clean array with count for ease of consumption by the next AI agent.  
  - Input: Aggregated URL data  
  - Output: JSON with array `urls` and `count`

- **Moodboard Generator Agent**  
  - Type: Langchain Agent  
  - Role: Receives title, description, and all 12 image URLs to produce a complete, self-contained two-page HTML moodboard.  
  - System Message: Detailed instructions to create a print-ready, styled HTML with two pages: page 1 visual collage and page 2 administrative summary including design details, colors, materials, and project metadata.  
  - AI Model: Uses Google Gemini Chat Model1  
  - Output: Raw HTML string  
  - Edge Cases: Large HTML output size, formatting errors, AI service errors.

- **Binary Converter**  
  - Type: Code node  
  - Role: Converts the HTML moodboard into a base64-encoded binary file named `index.html` for PDF generation compatibility.  
  - Input: HTML from Moodboard Generator Agent  
  - Output: Binary data with proper MIME and filename for Gotenberg  
  - Edge Cases: Missing or invalid HTML content.

- **PDF creator**  
  - Type: HTTP Request  
  - Role: Sends the HTML file binary to a Gotenberg instance (PDF rendering service) that produces a high-quality PDF with correct page breaks and A4 sizing.  
  - Authentication: HTTP Basic Auth  
  - Input: Binary HTML file  
  - Output: PDF binary file  
  - Edge Cases: Timeout, service unavailability, malformed input.

- **Send PDF**  
  - Type: Gmail node  
  - Role: Sends the generated PDF as an email attachment to the user’s email from the form input.  
  - Authentication: Gmail OAuth2  
  - Input: PDF binary from PDF creator, recipient email and subject from form data  
  - Edge Cases: Email send failures, invalid email addresses.

- **Sticky Notes** throughout this block explain each sub-process for image generation, uploading, public link creation, URL aggregation, HTML generation, PDF conversion, and email delivery.

---

### 3. Summary Table

| Node Name               | Node Type                             | Functional Role                                     | Input Node(s)                  | Output Node(s)                 | Sticky Note                                            |
|-------------------------|-------------------------------------|----------------------------------------------------|-------------------------------|-------------------------------|--------------------------------------------------------|
| Moodbaord Form          | formTrigger                         | Captures moodboard request input                    | —                             | Email extractor               | Explains form purpose and fields                        |
| Email extractor         | Code                               | Extracts local part of email for folder naming     | Moodbaord Form                | Create a folder               | Explains email extraction logic                         |
| Create a folder         | Nextcloud                          | Creates user folder in Nextcloud                    | Email extractor              | Conceptualization Agent       | Explains folder creation purpose                         |
| Think2                  | Langchain toolThink                | Internal AI conceptual analysis step                | Conceptualization Agent       | Conceptualization Agent       |                                                        |
| OpenRouter Chat Model   | Langchain LM Chat (Claude)         | AI language model for conceptualization             | Conceptualization Agent       | Conceptualization Agent       |                                                        |
| Simple Memory           | Langchain memoryBufferWindow       | Maintains session context                            | Conceptualization Agent       | Conceptualization Agent       |                                                        |
| Structured Output Parser2| Langchain Output Parser Structured| Parses AI JSON output into structured prompts       | Conceptualization Agent       | Concept Splitter              |                                                        |
| Conceptualization Agent | Langchain Agent                   | Generates 12 detailed image prompts in JSON         | Create a folder               | Concept Splitter              | Explains conceptualization agent role                   |
| Concept Splitter        | splitOut                          | Splits JSON object into individual prompts          | Conceptualization Agent       | Loop Over Items               | Explains splitting of image prompts                     |
| Loop Over Items         | splitInBatches                    | Loops over each prompt for image generation          | Concept Splitter              | Image Generator, URL Aggregate| Explains image prompt processing loop                   |
| Image Generator         | HTTP Request                      | Sends prompts to Hugging Face image generation API  | Loop Over Items               | Upload Image                 | Explains Hugging Face image generation                   |
| Upload Image            | Nextcloud                        | Uploads generated image to user folder               | Image Generator              | Share a file                 | Explains image uploading to Nextcloud                   |
| Share a file            | Nextcloud                        | Generates public share link for uploaded image       | Upload Image                 | set image title and url      |                                                        |
| set image title and url | Set                              | Sets image title and share URL for aggregation       | Share a file                 | Loop Over Items              |                                                        |
| URL Aggregate           | Aggregate                        | Aggregates all image URLs into a single list          | Loop Over Items              | Clean URLs                   | Explains URL aggregation                                |
| Clean URLs              | Code                            | Extracts and cleans URLs array for next step          | URL Aggregate               | Moodboard Generator Agent    | Explains URL extraction for email integration          |
| Moodboard Generator Agent| Langchain Agent                 | Generates two-page HTML moodboard from inputs         | Clean URLs                  | Binary Converter             | Explains two-page moodboard generation                  |
| Binary Converter        | Code                            | Converts HTML to base64 binary for PDF generation     | Moodboard Generator Agent   | PDF creator                 | Explains HTML to binary conversion                       |
| PDF creator             | HTTP Request                    | Sends HTML to Gotenberg for PDF rendering              | Binary Converter            | Send PDF                    | Explains PDF generation                                 |
| Send PDF                | Gmail                          | Sends final PDF moodboard to user email                 | PDF creator                 | —                           | Explains email delivery of PDF                          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Moodboard Input Form:**  
   - Add a **Form Trigger** node.  
   - Title: "Moodboard"  
   - Fields:  
     - Title (text, required)  
     - Description (textarea, required)  
     - Email (email, required)  
   - Enable webhook to receive submissions.

2. **Extract Email Local Part:**  
   - Add a **Code** node named "Email extractor".  
   - JS code: Extract local part before '@', remove '+' tags.  
   - Input: Use form JSON email field.

3. **Create User Folder in Nextcloud:**  
   - Add a **Nextcloud** node "Create a folder".  
   - Operation: Create folder at `/moodboard/{{ $json.localPart }}` using extracted local part.  
   - Credentials: Configure Nextcloud API with folder creation rights.

4. **Set up AI Conceptualization Agent:**  
   - Add **Langchain Agent** node "Conceptualization Agent".  
   - Input text uses title and description from form.  
   - System prompt: Detailed instructions to generate 12 image prompts in JSON, including a 3D view prompt.  
   - Connect agent to AI language models: OpenRouter Claude and optionally Google Gemini for better context.  
   - Add **Simple Memory** node for session context keyed by title.  
   - Add **Structured Output Parser** node configured with JSON schema example for the 12 prompts.  
   - Connect parser output to a **splitOut** node "Concept Splitter" to separate 12 prompts.

5. **Image Generation Loop:**  
   - Add **splitInBatches** node "Loop Over Items" to process each prompt.  
   - Inside loop:  
     - **HTTP Request** node "Image Generator" to send prompt text to Hugging Face API model "black-forest-labs/FLUX.1-schnell".  
       - Use HTTP header auth with Hugging Face token.  
     - **Nextcloud** node "Upload Image" to upload image binary to `/moodboard/{{localPart}}/{{title}}.jpg`.  
     - **Nextcloud** node "Share a file" to create public share link for the uploaded image.  
     - **Set** node "set image title and url" to store title and public URL with `/preview` suffix.

6. **Aggregate URLs:**  
   - Use **Aggregate** node "URL Aggregate" to combine all URLs from loop results.  
   - Use **Code** node "Clean URLs" to extract URL array and count.

7. **Generate Two-Page Moodboard HTML:**  
   - Add **Langchain Agent** node "Moodboard Generator Agent" using Google Gemini Chat Model.  
   - Input: JSON including title, description, and array of 12 URLs.  
   - System prompt: Instructions to produce print-ready two-page HTML moodboard with specified layout and style rules.

8. **Convert HTML to Binary:**  
   - Add **Code** node "Binary Converter".  
   - Convert HTML string to UTF-8 buffer, encode base64, and set filename to `index.html` for Gotenberg compatibility.

9. **PDF Generation:**  
   - Add **HTTP Request** node "PDF creator".  
   - POST multipart/form-data with binary HTML file to Gotenberg PDF rendering service.  
   - Use HTTP Basic Auth credentials.

10. **Email Delivery:**  
    - Add **Gmail** node "Send PDF".  
    - Send email to user’s email from form input with PDF attached.  
    - Configure Gmail OAuth2 credentials.

11. **Connect nodes in the order described above**, ensuring correct data flow:  
    Form Trigger → Email extractor → Create folder → Conceptualization Agent → Structured Output Parser → Concept Splitter → Loop Over Items → Image Generator → Upload Image → Share a file → set image title and url → URL Aggregate → Clean URLs → Moodboard Generator Agent → Binary Converter → PDF creator → Send PDF.

12. **Add Sticky Notes** at key points to explain node purposes, especially for complex AI prompts and API interactions.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     | Context or Link                                                                                                        |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------|
| This workflow uses advanced AI agents to generate highly detailed image prompts and HTML moodboard layouts, leveraging Langchain integration with Google Gemini and OpenRouter’s Claude. The final output is a professional, print-ready two-page PDF moodboard automatically emailed to the user.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                | Workflow description and AI integration details                                                                        |
| Hugging Face API is used to convert textual image prompts into high-quality JPEG images using the model "black-forest-labs/FLUX.1-schnell". Authenticate with an API token.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         | Hugging Face model for image generation                                                                                |
| Nextcloud is used as a secure, structured storage for generated images, with personalized folders per user. Public sharing links are generated to embed images in the moodboard and emails.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   | Nextcloud API usage for file storage and sharing                                                                       |
| Gotenberg service renders HTML to PDF with precise A4 sizing and print-quality output. Requires multipart form data upload and basic authentication.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         | PDF generation service                                                                                                 |
| Email delivery uses Gmail OAuth2 credentials. The workflow attaches the generated PDF and sends it to the user with the original description as message content.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   | Gmail node for email sending                                                                                           |
| The detailed AI prompt instructions for the Conceptualization Agent and Moodboard Generator Agent ensure high fidelity to interior design professional standards, including style guides, color palettes, and image composition details.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            | AI prompt engineering for interior design moodboard generation                                                       |
| The workflow is designed to handle errors gracefully: Nextcloud folder creation and image upload nodes continue on failure to avoid blocking the whole flow. AI nodes require proper API quotas and error monitoring.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 | Error handling considerations                                                                                          |
| Test mode is supported by the form field "formMode" to distinguish between live and test submissions.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  | Form input management and testing                                                                                      |
| Workflow requires multiple API credentials setup: Google Palm API (Gemini), OpenRouter API (Claude), Hugging Face API, Nextcloud API, Gotenberg HTTP Basic Auth, Gmail OAuth2.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                | Credential setup instructions                                                                                          |
| Workflow layout and sticky notes provide detailed inline documentation for maintainers and advanced users to understand and modify each step easily.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 | Inline documentation and maintainability                                                                               |

---

**Disclaimer:** The provided textual content is derived solely from an n8n automated workflow. It fully complies with applicable content policies and contains no illegal or offensive material. All processed data is legal and public.

# End of Document