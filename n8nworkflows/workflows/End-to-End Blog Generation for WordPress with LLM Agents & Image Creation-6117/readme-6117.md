End-to-End Blog Generation for WordPress with LLM Agents & Image Creation

https://n8nworkflows.xyz/workflows/end-to-end-blog-generation-for-wordpress-with-llm-agents---image-creation-6117


# End-to-End Blog Generation for WordPress with LLM Agents & Image Creation

### 1. Workflow Overview

This workflow automates end-to-end blog post generation tailored for WordPress, focusing on insurance claim-related content. It leverages a multi-agent architecture using large language model (LLM) agents to perform structured content generation, editing, metadata creation, and image prompt generation, culminating in automated WordPress post creation with optional featured image upload.

The workflow is designed for users wanting high-quality, SEO-optimized, authoritative blog posts that serve homeowners, business owners, or policyholders seeking insurance claim advice. It accepts user input from a form and orchestrates content creation through specialized AI agents and sub-workflows, finishing with WordPress publishing and optional image handling.

**Logical Blocks:**

- **1.1 Input Reception & Data Setup:** Receives user input from form submission and initializes workflow variables.
- **1.2 Online Information Retrieval:** Gathers real-time web information relevant to keywords.
- **1.3 Orchestration Agent:** Coordinates the content creation pipeline, invoking multiple specialized sub-agents/tools in sequence.
- **1.4 Content Generation Sub-Agents:** Includes OutlinePlanner, createSections, SectionWriter, Editor, and MetaInfo agents to progressively build and refine blog content.
- **1.5 Image Prompt and Generation:** Creates an AI prompt for a cover image, generates the image, resizes it, uploads it to WordPress, and updates media metadata.
- **1.6 WordPress Posting:** Posts the generated blog draft and sets the featured image if applicable.
- **1.7 Readiness Check & Error Handling:** Determines if content is ready for publishing and stops execution with an error if not.

---

### 2. Block-by-Block Analysis

---

#### 2.1 Input Reception & Data Setup

**Overview:**  
Initial block triggered by a form submission that collects blog generation parameters and sets up workflow variables accordingly.

**Nodes Involved:**  
- On form submission  
- db

**Node Details:**

- **On form submission**  
  - *Type:* Form Trigger  
  - *Role:* Entry point capturing user inputs including keywords, word count, number of sections, image generation flag, writing style, and website URL.  
  - *Config:* Form fields for KeyWords (required), wordCount, Number of sections, Generate a featured image (dropdown: true/false), Writing Style, Web Site (required).  
  - *Output:* Passes collected form data downstream.

- **db**  
  - *Type:* Set  
  - *Role:* Extracts and maps form inputs into named variables for use across the workflow.  
  - *Config:* Assigns variables like website URL, authors (fixed array [6,7]), keywords, wordCount, number_of_sections, create_image flag, Writing Style, aboutWebsite (concatenation of website URL + "/about").  
  - *Input:* From form submission node.  
  - *Output:* Provides structured data for subsequent nodes.

**Edge Cases & Failures:**  
- Missing required form inputs (e.g., Keywords or Website) halts workflow start.  
- Invalid URL format in Website input could lead to downstream HTTP request failures.

---

#### 2.2 Online Information Retrieval

**Overview:**  
Retrieves real-time, SEO-relevant questions and answers from the web based on keywords to enhance content accuracy and authority.

**Nodes Involved:**  
- GetOnilneInfo  
- OpenRouter Chat Model

**Node Details:**

- **GetOnilneInfo**  
  - *Type:* LangChain Agent  
  - *Role:* Executes a specialized prompt to generate the top 5 commonly searched insurance claim questions and answers using real-time web data, structured in JSON.  
  - *Config:* Uses keywords and aboutWebsite to contextualize search. System message instructs agent to produce verified, cited answers with URLs.  
  - *Input:* JSON containing keywords and aboutWebsite from `db` node.  
  - *Output:* Structured JSON of questions & answers with sources.

- **OpenRouter Chat Model**  
  - *Type:* Language Model (OpenRouter)  
  - *Role:* Provides the LLM backend for the GetOnilneInfo agent.  
  - *Config:* Uses "perplexity/sonar:online" model for real-time data integration.  
  - *Credential:* OpenRouter account.  
  - *Input/Output:* Connected internally to GetOnilneInfo.

**Edge Cases & Failures:**  
- API rate limits or network errors from OpenRouter may cause retrieval failure.  
- If keywords are too broad or irrelevant, output may lack quality or relevance.

---

#### 2.3 Orchestration Agent

**Overview:**  
The central coordinator node managing the entire content generation pipeline. It parses inputs, invokes sub-agent tools in a strict order, and consolidates outputs.

**Nodes Involved:**  
- OrchestrationAgent  
- Structured Output Parser1  
- Check ready for publish  
- If  
- Not Ready

**Node Details:**

- **OrchestrationAgent**  
  - *Type:* LangChain Agent (Orchestration multi-agent)  
  - *Role:* Directs the workflow, calling sub-agents in sequence: OutlinePlanner → createSections → SectionWriter → Editor → MetaInfo → ImagePrompt.  
  - *Config:* Uses a detailed system prompt describing each step and data to be passed. Inputs include keywords, number_of_sections, search results from GetOnilneInfo, writing style.  
  - *Inputs:* Aggregated from `db` and `GetOnilneInfo`.  
  - *Outputs:* JSON with final blog content, table of contents, metadata, image prompt data, and a readiness flag.  
  - *Retry:* Up to 5 tries on failure.  
  - *Output Parser:* Uses Structured Output Parser1 to parse JSON output.

- **Structured Output Parser1**  
  - *Type:* LangChain Output Parser  
  - *Role:* Validates and parses OrchestrationAgent’s JSON output to ensure correct structure for downstream nodes.  
  - *Schema:* Validates fields such as `ready` (boolean), `table_of_content` (string), `final_blog_post_html` (string), metadata, and image data.

- **Check ready for publish**  
  - *Type:* If Node  
  - *Role:* Checks if the OrchestrationAgent output signals readiness (`ready: true`).  
  - *Outputs:* Routes to either `If` node (true) or `Not Ready` node (false).

- **If**  
  - *Type:* If Node  
  - *Role:* Further checks if featured image generation is requested (`create_image` flag).  
  - *Outputs:* If true, routes to image generation; if false, routes directly to post publishing.

- **Not Ready**  
  - *Type:* Stop and Error  
  - *Role:* Stops workflow with error message if content is not ready for publishing.

**Edge Cases & Failures:**  
- Expression or parsing errors if OrchestrationAgent fails or returns malformed JSON.  
- Logic errors if readiness flag is missing or false.  
- Must handle retry logic on OrchestrationAgent failures.

---

#### 2.4 Content Generation Sub-Agents

**Overview:**  
These nodes implement the multi-step content creation pipeline. Each node is a sub-workflow or agent specializing in one stage: outline planning, section creation, section writing, editing, and metadata generation.

**Nodes Involved:**  
- OutlinePlanner (toolWorkflow)  
- createSections (toolWorkflow)  
- SectionWriter (toolWorkflow)  
- Editor (toolWorkflow)  
- metaInfo (toolWorkflow)  
- AI Agent (content outline strategist)  
- AI Agent1 (createSections agent)  
- AI Agent2 (SectionWriter agent)  
- AI Agent3 (Editor agent)  
- AI Agent4 (MetaInfo agent)  
- OpenAI Chat Models (multiple) for underlying LLM calls

**Node Details:**

- **OutlinePlanner**  
  - *Type:* LangChain toolWorkflow  
  - *Role:* Generates a structured, SEO-optimized table of contents (ToC) for the blog post based on input query.  
  - *Input:* Topic, keywords, number_of_sections, style, tone from orchestration agent.  
  - *Sub-Workflow:* Calls a dedicated outline planner sub-workflow by ID.  
  - *Output:* ToC JSON for next step.

- **createSections**  
  - *Type:* LangChain toolWorkflow  
  - *Role:* Converts the ToC into detailed blog sections with titles and short descriptions.  
  - *Input:* ToC JSON.  
  - *Sub-Workflow:* Calls createSections sub-workflow by ID.  
  - *Output:* JSON list of section titles and descriptions.

- **SectionWriter**  
  - *Type:* LangChain toolWorkflow  
  - *Role:* Writes full content for each blog section, ensuring SEO and legal accuracy.  
  - *Input:* Sections from createSections node, plus style/tone.  
  - *Sub-Workflow:* Calls SectionWriter sub-workflow by ID.

- **Editor**  
  - *Type:* LangChain toolWorkflow  
  - *Role:* Edits and combines section content into a polished final blog post.  
  - *Input:* SectionWriter output plus number_of_sections, style, tone.  
  - *Sub-Workflow:* Editor sub-workflow by ID.

- **metaInfo**  
  - *Type:* LangChain toolWorkflow  
  - *Role:* Generates blog post metadata including title, slug, and description for SEO and WordPress.  
  - *Input:* Final post, topic, style, tone, keywords.  
  - *Sub-Workflow:* MetaInfo sub-workflow by ID.

- **AI Agents and OpenAI Chat Models**  
  - Used internally by sub-workflows to execute LLM calls with specific prompts, such as GPT-4.1 or GPT-4o-mini models.  
  - Agents have specialized system messages to guide content structuring and writing.

**Edge Cases & Failures:**  
- Sub-workflow ID mismatches or missing workflows cause failures.  
- LLM API rate limits or quota exhaustion.  
- Parsing or schema validation errors between steps.  
- Content quality depends on prompt design and LLM stability.

---

#### 2.5 Image Prompt and Generation

**Overview:**  
If requested, generates an image prompt, creates a cover image via OpenAI, resizes the image, uploads to WordPress media library, and updates media metadata.

**Nodes Involved:**  
- ImagePrompt (toolWorkflow)  
- AI Agent5 (Image prompt assistant)  
- Structured Output Parser  
- OpenAI Chat Model7  
- Generate Featured Image (OpenAI Image generation)  
- Resize Image  
- Upload Image To WP  
- Update Meta Data1  
- Set Featured Image

**Node Details:**

- **ImagePrompt**  
  - *Type:* LangChain toolWorkflow  
  - *Role:* Generates an AI prompt and alt text for a cover image based on the blog title and keywords.  
  - *Sub-Workflow:* Calls ImagePrompt sub-workflow by ID.

- **AI Agent5**  
  - *Type:* LangChain Agent  
  - *Role:* Specifically crafts the image prompt and alt text.  
  - *Output Parser:* Uses Structured Output Parser to validate image data.

- **Generate Featured Image**  
  - *Type:* OpenAI Image Generation Node  
  - *Role:* Requests a photographic, realistic image based on the prompt.  
  - *Config:* 1024x1024 size, natural style, no URL return (image binary).  
  - *Credential:* OpenAI API.

- **Resize Image**  
  - *Type:* Image Edit Node  
  - *Role:* Resizes the generated image as needed before upload.

- **Upload Image To WP**  
  - *Type:* HTTP Request (WordPress REST API)  
  - *Role:* Uploads the binary image to WordPress media library with correct headers.  
  - *Credential:* WordPress API credentials.

- **Update Meta Data1**  
  - *Type:* HTTP Request (WordPress REST API)  
  - *Role:* Updates media metadata with title, slug, alt text, caption, and description using blog metadata.  
  - *Credential:* WordPress API.

- **Set Featured Image**  
  - *Type:* HTTP Request (WordPress REST API)  
  - *Role:* Sets the uploaded media as the featured image of the newly created blog post by updating the post's featured_media field.  
  - *Credential:* WordPress API.

**Edge Cases & Failures:**  
- OpenAI image generation failures or deprecated models.  
- WordPress API authentication failures or permission errors.  
- Image upload failures due to network or API limits.  
- Missing or malformed image prompt could cause poor image output.

---

#### 2.6 WordPress Posting

**Overview:**  
Posts the finalized blog content as a draft on WordPress with associated metadata and author assignment.

**Nodes Involved:**  
- Post Blog To WP  
- Post Blog To WP1

**Node Details:**

- **Post Blog To WP**  
  - *Type:* WordPress node (via API)  
  - *Role:* Creates a new draft post with title, slug, HTML content, assigned author (randomly chosen from authors array), and category (fixed to 2).  
  - *Credential:* WordPress API credentials.  
  - *Input:* Metadata and final blog HTML from the orchestration output.

- **Post Blog To WP1**  
  - *Type:* WordPress node (duplicate of above)  
  - *Role:* Used when no featured image is generated; posts the blog draft directly.  
  - *Credential:* Same WordPress API.

**Edge Cases & Failures:**  
- WordPress API permission errors or invalid credentials.  
- Invalid HTML content or metadata causing post creation failure.  
- Author ID selection must match existing WordPress users.

---

#### 2.7 Readiness Check & Error Handling

**Overview:**  
Verifies if blog content generation succeeded and routes the workflow appropriately.

**Nodes Involved:**  
- Check ready for publish  
- If  
- Not Ready

**Node Details:**

- **Check ready for publish**  
  - *Type:* If Node  
  - *Role:* Checks readiness boolean from OrchestrationAgent output.  
  - *Output:* Routes to image generation or posting, or stops with error.

- **Not Ready**  
  - *Type:* Stop and Error Node  
  - *Role:* Stops workflow execution with a message "The content not ready for publish" if readiness is false.

**Edge Cases & Failures:**  
- False negative readiness due to partial failures in sub-agents.  
- Workflow stops without posting content if not ready.

---

### 3. Summary Table

| Node Name             | Node Type                              | Functional Role                               | Input Node(s)                 | Output Node(s)                  | Sticky Note                                           |
|-----------------------|--------------------------------------|-----------------------------------------------|------------------------------|--------------------------------|-------------------------------------------------------|
| On form submission     | Form Trigger                         | Entry point: receives user blog post inputs   | -                            | db                             |                                                       |
| db                    | Set                                  | Maps form inputs into workflow variables       | On form submission            | GetOnilneInfo                  |                                                       |
| GetOnilneInfo          | LangChain Agent                      | Retrieves SEO Q&A via real-time web search     | db                           | OrchestrationAgent             | ## Get Online Info                                     |
| OpenRouter Chat Model  | LangChain LM Chat Model              | Provides LLM backend for GetOnilneInfo         | GetOnilneInfo                | GetOnilneInfo                  |                                                       |
| OrchestrationAgent     | LangChain Agent (Orchestration)     | Coordinates multi-agent content generation     | db, GetOnilneInfo            | Check ready for publish        | # Generate the content                                 |
| Structured Output Parser1 | LangChain Output Parser          | Parses OrchestrationAgent output JSON          | OrchestrationAgent           | Check ready for publish        |                                                       |
| Check ready for publish| If                                  | Checks content readiness flag                   | OrchestrationAgent           | If, Not Ready                  |                                                       |
| If                    | If                                  | Checks if featured image creation requested    | Check ready for publish       | Generate Featured Image, Post Blog To WP1 |                                                       |
| Not Ready             | Stop and Error                      | Stops workflow if content not ready             | Check ready for publish       | -                              |                                                       |
| OutlinePlanner         | LangChain toolWorkflow               | Generates blog post Table of Contents           | OrchestrationAgent           | OrchestrationAgent             | # Outline Planner                                     |
| createSections         | LangChain toolWorkflow               | Creates detailed blog sections from ToC         | OrchestrationAgent           | OrchestrationAgent             | # Create sections                                    |
| SectionWriter          | LangChain toolWorkflow               | Writes content for each blog section             | OrchestrationAgent           | OrchestrationAgent             | # Section Writer                                     |
| Editor                 | LangChain toolWorkflow               | Edits and combines sections into full post      | OrchestrationAgent           | OrchestrationAgent             | # Editor                                            |
| metaInfo               | LangChain toolWorkflow               | Generates title, slug, and description metadata | OrchestrationAgent           | OrchestrationAgent             | # Meta Info                                         |
| ImagePrompt            | LangChain toolWorkflow               | Generates AI prompt for cover image              | OrchestrationAgent           | AI Agent5                     | # Image Prompt                                      |
| AI Agent5              | LangChain Agent                     | Creates image prompt and alt text                 | ImagePrompt                  | Structured Output Parser        |                                                       |
| Structured Output Parser | LangChain Output Parser            | Parses image prompt JSON output                   | AI Agent5                   | Generate Featured Image         |                                                       |
| Generate Featured Image | OpenAI Image Generation            | Generates cover image from prompt                 | Structured Output Parser     | Resize Image                   | ## Generate Image                                   |
| Resize Image           | Image Edit                          | Resizes generated image                           | Generate Featured Image      | Upload Image To WP             |                                                       |
| Upload Image To WP     | HTTP Request                        | Uploads image to WordPress media library          | Resize Image                | Update Meta Data1              | ## Upload to wrodpress                              |
| Update Meta Data1      | HTTP Request                        | Updates WordPress media metadata                   | Upload Image To WP          | Post Blog To WP                |                                                       |
| Set Featured Image     | HTTP Request                        | Sets featured image on WordPress post               | Post Blog To WP             | -                             |                                                       |
| Post Blog To WP        | WordPress Node                     | Posts blog draft with metadata and content        | Update Meta Data1           | Set Featured Image             |                                                       |
| Post Blog To WP1       | WordPress Node                     | Posts blog draft without featured image            | If                         | -                             |                                                       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Form Trigger Node ("On form submission")**  
   - Type: Form Trigger  
   - Configure form titled "Wrodpress Post Creation" with fields:  
     - KeyWords (required, string)  
     - wordCount (integer)  
     - Number of sections (3-5, string)  
     - Generate a featured image (dropdown true/false)  
     - Writing Style (string)  
     - Web Site (required URL string)  

2. **Create Set Node ("db")**  
   - Type: Set  
   - Map form inputs to variables:  
     - website = Web Site  
     - authors = [6,7] (hardcoded author IDs)  
     - keywords = KeyWords  
     - wordCount = wordCount  
     - number_of_sections = Number of sections  
     - create_image = Generate a featured image  
     - Writing Style = Writing Style  
     - aboutWebsite = website + "/about"  
   - Connect "On form submission" → "db"

3. **Create LangChain Agent Node ("GetOnilneInfo")**  
   - Type: LangChain Agent  
   - System message: Expert SEO content writer prompt for generating top 5 questions and answers with sources based on keywords and aboutWebsite.  
   - Input: keywords and aboutWebsite from "db"  
   - Connect "db" → "GetOnilneInfo"

4. **Create OpenRouter Chat Model Node ("OpenRouter Chat Model")**  
   - Model: "perplexity/sonar:online"  
   - Credential: OpenRouter API key  
   - Connect as backend to "GetOnilneInfo"

5. **Create LangChain Agent Node ("OrchestrationAgent")**  
   - Type: Orchestration multi-agent  
   - System message: Detailed orchestration instructions including calling sub-agents in order: OutlinePlanner, createSections, SectionWriter, Editor, MetaInfo, ImagePrompt.  
   - Input: keywords, number_of_sections, search results from GetOnilneInfo, writing style from "db"  
   - Max tries: 5  
   - Connect "GetOnilneInfo" → "OrchestrationAgent"  
   - Connect "db" → "OrchestrationAgent"

6. **Create Structured Output Parser Node ("Structured Output Parser1")**  
   - Define JSON schema to parse OrchestrationAgent output (ready flag, final blog post HTML, ToC, metadata, image data)  
   - Connect "OrchestrationAgent" → "Structured Output Parser1"

7. **Create If Node ("Check ready for publish")**  
   - Condition: Check if output.ready == true from "Structured Output Parser1"  
   - Connect "Structured Output Parser1" → "Check ready for publish"

8. **Create If Node ("If")**  
   - Condition: If "db".create_image contains "true"  
   - Connect "Check ready for publish" true output → "If"

9. **Create Stop and Error Node ("Not Ready")**  
   - Message: "The content not ready for publish"  
   - Connect "Check ready for publish" false output → "Not Ready"

10. **Create toolWorkflow Nodes for content generation:**  
    - OutlinePlanner: Calls sub-workflow for ToC generation (provide workflow ID)  
    - createSections: Calls sub-workflow to create detailed sections (ID)  
    - SectionWriter: Calls sub-workflow to write sections (ID)  
    - Editor: Calls sub-workflow to edit and combine content (ID)  
    - metaInfo: Calls sub-workflow for metadata generation (ID)  
    - ImagePrompt: Calls sub-workflow to generate image prompt (ID)  
    - Connect all these toolWorkflows as ai_tool outputs to "OrchestrationAgent"

11. **Create AI Agents and OpenAI Chat Model Nodes**  
    - Create AI agents with system prompts for OutlinePlanner, createSections, SectionWriter, Editor, MetaInfo, and ImagePrompt as per the workflow, connected internally to respective toolWorkflows.  
    - Use OpenAI GPT-4.1 or GPT-4o-mini models with OpenAI credentials.  
    - Connect each AI agent with its respective OpenAI model node.

12. **Create LangChain Agent Node ("AI Agent5") for Image Prompt**  
    - Specialized prompt to generate image prompt and alt text from blog title  
    - Connect "ImagePrompt" → "AI Agent5"

13. **Create Structured Output Parser Node ("Structured Output Parser")**  
    - Parse image prompt JSON output from "AI Agent5"  
    - Connect "AI Agent5" → "Structured Output Parser"

14. **Create OpenAI Image Generation Node ("Generate Featured Image")**  
    - Prompt: Use image prompt and blog title from "Check ready for publish" output  
    - Size: 1024x1024, style: natural  
    - Credentials: OpenAI API key  
    - Connect "If" true output → "Generate Featured Image"

15. **Create Image Edit Node ("Resize Image")**  
    - Operation: Resize image (default or custom settings)  
    - Connect "Generate Featured Image" → "Resize Image"

16. **Create HTTP Request Node ("Upload Image To WP")**  
    - Method: POST  
    - URL: {website} + "wp-json/wp/v2/media"  
    - Authentication: WordPress API credentials  
    - Headers: content-disposition with filename, content-type from binary data  
    - Input: binary image from "Resize Image"  
    - Connect "Resize Image" → "Upload Image To WP"

17. **Create HTTP Request Node ("Update Meta Data1")**  
    - Method: POST  
    - URL: {website} + "wp-json/wp/v2/media/{media_id}"  
    - Body parameters: title, slug, alt_text, caption, description from "Check ready for publish" output  
    - Connect "Upload Image To WP" → "Update Meta Data1"

18. **Create WordPress Node ("Post Blog To WP")**  
    - Action: Create post  
    - Title, slug, content, authorId (random from authors array), category = 2  
    - Status: draft  
    - Credentials: WordPress API  
    - Connect "Update Meta Data1" → "Post Blog To WP"

19. **Create HTTP Request Node ("Set Featured Image")**  
    - Method: POST  
    - URL: {website} + "wp-json/wp/v2/posts/{post_id}"  
    - Body parameter: featured_media = media ID from "Update Meta Data1"  
    - Connect "Post Blog To WP" → "Set Featured Image"

20. **Create WordPress Node ("Post Blog To WP1")**  
    - Same as "Post Blog To WP" but for cases when no featured image is generated  
    - Connect "If" false output → "Post Blog To WP1"

---

### 5. General Notes & Resources

| Note Content                                                                                                                     | Context or Link                                      |
|---------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------|
| This workflow uses a multi-agent orchestration pattern with LangChain agents to modularize blog content creation steps.          | Design pattern for scalable LLM workflows           |
| The content is specialized for insurance claims, targeting SEO-rich, authoritative blog posts for WordPress.                     | Use case focus                                       |
| The workflow includes retry logic (max 5 tries) on the orchestration agent to handle transient LLM or API errors.                | Reliability best practice                            |
| Featured image generation is optional and uses OpenAI’s image generation with controlled style parameters for realism.           | Image generation best practices                      |
| WordPress REST API integration requires an API credential with permissions for posts and media management.                       | WordPress API docs: https://developer.wordpress.org/rest-api/ |
| Prompts include specific instructions for SEO, legal accuracy, inline HTML citations, and internal linking for high-quality content. | Content guidelines                                   |
| Sticky notes grouped nodes by function for clarity, e.g., "# Generate the content", "# Outline Planner", "# Upload to wrodpress". | Visual workflow documentation aid                    |

---

**Disclaimer:** The provided text is exclusively derived from an automated n8n workflow and complies with applicable content policies. It contains no illegal, offensive, or protected materials. All data processed is lawful and publicly accessible.