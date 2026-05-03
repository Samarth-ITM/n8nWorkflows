Create Blog Posts from YouTube Videos with Mistral AI & Gemini Image Generation

https://n8nworkflows.xyz/workflows/create-blog-posts-from-youtube-videos-with-mistral-ai---gemini-image-generation-8190


# Create Blog Posts from YouTube Videos with Mistral AI & Gemini Image Generation

### 1. Workflow Overview

This workflow automates the creation of blog posts from YouTube videos by extracting video transcripts, analyzing content with AI (Mistral AI and Ollama models), generating structured blog sections, and producing images with Gemini image generation. It then publishes the blog post on WordPress with optimized metadata and featured images.

The workflow is logically divided into the following blocks:

- **1.1 YouTube Content Acquisition:** Detects new videos from a YouTube channel RSS feed and retrieves the video's transcript.
- **1.2 Content Cleaning and Intent Analysis:** Cleans the raw transcript and analyzes the content’s intent and style using AI models.
- **1.3 Blog Post Structure Generation:** Creates a structured outline with sections and table of contents based on the analyzed content.
- **1.4 Blog Content Generation:** Generates detailed blog content sections, performs EEAT (Expertise, Authoritativeness, Trustworthiness) enhancement, and edits content.
- **1.5 Image Generation and Upload:** Generates image prompts from content, creates images using Gemini, resizes, uploads to WordPress, and sets featured images.
- **1.6 Blog Metadata and Publishing:** Generates blog metadata, converts content to file formats, creates or updates WordPress posts, and assigns featured images.

---

### 2. Block-by-Block Analysis

#### 1.1 YouTube Content Acquisition

**Overview:**  
This block monitors a YouTube channel via RSS feed for new video uploads, retrieves the video transcript through HTTP requests, and prepares data for further processing.

**Nodes Involved:**  
- Check Youtube Channel (RSS Feed Trigger)  
- Get Transcript (HTTP Request)  
- Clean Data (Code)

**Node Details:**

- **Check Youtube Channel**  
  - Type: RSS Feed Read Trigger  
  - Role: Triggers the workflow on new YouTube videos by polling RSS feed.  
  - Configuration: RSS feed URL of the YouTube channel (not shown explicitly).  
  - Input: None (trigger node)  
  - Output: Video metadata items  
  - Edge Cases: RSS feed unavailability, network timeout, empty feed.

- **Get Transcript**  
  - Type: HTTP Request  
  - Role: Requests transcript data for the new video (likely from a transcript API or service).  
  - Configuration: API endpoint and parameters to fetch transcript (not detailed here).  
  - Input: Video metadata from RSS trigger  
  - Output: Raw transcript text  
  - Edge Cases: Transcript API errors, missing transcripts, latency.

- **Clean Data**  
  - Type: Code  
  - Role: Cleans and preprocesses raw transcript text for AI processing (removes noise, formatting).  
  - Configuration: Custom JavaScript code (not detailed)  
  - Input: Transcript text  
  - Output: Cleaned transcript  
  - Edge Cases: Unexpected transcript formats, code execution errors.

---

#### 1.2 Content Cleaning and Intent Analysis

**Overview:**  
Analyzes the cleaned transcript to determine the style and intent of the content using AI, setting the context for blog creation.

**Nodes Involved:**  
- Search intent and style (LangChain Agent)  
- Structured Output Parser1 (Output Parser)  
- Ollama Chat Model (LM Chat Ollama for AI inference)

**Node Details:**

- **Search intent and style**  
  - Type: LangChain Agent (AI)  
  - Role: Uses AI to analyze transcript style and intent (e.g., tone, target audience).  
  - Configuration: Connected to Ollama Chat Model for language model inference.  
  - Input: Cleaned transcript  
  - Output: Structured intent and style data  
  - Edge Cases: AI model failures, ambiguous content, prompt misinterpretation.

- **Structured Output Parser1**  
  - Type: Output Parser (LangChain)  
  - Role: Parses AI output into structured JSON or defined format for downstream nodes.  
  - Input: AI response from Search intent and style  
  - Output: Parsed structured data  
  - Edge Cases: Parsing errors, malformed AI output.

- **Ollama Chat Model**  
  - Type: Language Model Chat Node (Ollama)  
  - Role: Provides AI inference for intent and style detection.  
  - Configuration: Model endpoint, parameters (not detailed)  
  - Input: Prompt from Search intent and style node  
  - Output: AI-generated text  
  - Edge Cases: API unavailability, rate limits, model errors.

---

#### 1.3 Blog Post Structure Generation

**Overview:**  
Generates a detailed blog post outline including sections and table of contents from the analyzed intent and style data.

**Nodes Involved:**  
- Table of Contents (LangChain Agent)  
- Structured Output Parser (Output Parser)  
- Create the Sections (LangChain Agent)  
- Split Out1 (Split Out node)  
- Merge (Merge node)

**Node Details:**

- **Table of Contents**  
  - Type: LangChain Agent  
  - Role: Creates a table of contents based on blog structure and content intent.  
  - Input: Parsed intent/style data  
  - Output: Table of contents text or structure  
  - Connected to Create the Sections node.

- **Structured Output Parser**  
  - Type: LangChain Output Parser  
  - Role: Parses AI output from Table of Contents into structured format.  
  - Input: AI output from Table of Contents  
  - Output: Structured sections data  

- **Create the Sections**  
  - Type: LangChain Agent  
  - Role: Generates detailed blog sections content based on table of contents and input data.  
  - Input: Parsed table of contents  
  - Output: Sections content for blog post  
  - Output connected to Split Out1.

- **Split Out1**  
  - Type: Split Out  
  - Role: Splits the generated sections into separate items for parallel processing or aggregation.  
  - Input: Sections content  
  - Output: Splits content to Generate the Content and Merge nodes.

- **Merge**  
  - Type: Merge  
  - Role: Recombines processed sections into a single flow.  
  - Input: One input from Split Out1 and another from Generate the Content  
  - Output: Merged content for aggregation.

---

#### 1.4 Blog Content Generation

**Overview:**  
Generates detailed content for each blog section, enriches it with EEAT standards, edits content, and prepares the final draft.

**Nodes Involved:**  
- Generate the Content (LangChain Agent)  
- Aggregate1 (Aggregate node)  
- EEAT (LangChain Agent)  
- Content Editor (LangChain Agent)  
- Title Maker1 (LangChain Agent)

**Node Details:**

- **Generate the Content**  
  - Type: LangChain Agent  
  - Role: Produces detailed text content for each blog section.  
  - Input: Sections split from Split Out1  
  - Output: Generated content chunks  

- **Aggregate1**  
  - Type: Aggregate  
  - Role: Aggregates generated content chunks into a single dataset.  
  - Input: Output from Merge node  
  - Output: Aggregated content  

- **EEAT**  
  - Type: LangChain Agent  
  - Role: Enhances content with Expertise, Authoritativeness, and Trustworthiness attributes.  
  - Input: Aggregated content  
  - Output: EEAT-enhanced content  

- **Content Editor**  
  - Type: LangChain Agent  
  - Role: Edits and polishes content for clarity and consistency.  
  - Input: EEAT-enhanced content  
  - Output: Edited blog content  

- **Title Maker1**  
  - Type: LangChain Agent  
  - Role: Generates an engaging blog title based on edited content.  
  - Input: Edited content  
  - Output: Blog title  

---

#### 1.5 Image Generation and Upload

**Overview:**  
Generates image prompts, creates images via Gemini AI, resizes images, uploads them to WordPress, and sets featured images for the blog post.

**Nodes Involved:**  
- Generate Image Prompt + Alt Text (LangChain Agent)  
- Structured Output Parser3 (Output Parser)  
- Nano 🍌 (HTTP Request) — presumably Gemini image generation service  
- Edit Fields1 (Set node)  
- Convert to File (Convert to File node)  
- Resize Image (Edit Image)  
- Upload Image To WP (HTTP Request)  
- Create a post (WordPress node)  
- Update a post (WordPress node)  
- Set Featured Image (HTTP Request)

**Node Details:**

- **Generate Image Prompt + Alt Text**  
  - Type: LangChain Agent  
  - Role: Creates image generation prompts and alt text from blog content.  
  - Input: Edited content or metadata  
  - Output: Image prompt and alt text  

- **Structured Output Parser3**  
  - Type: Output Parser  
  - Role: Parses image prompt and alt text AI output into structured format.  

- **Nano 🍌**  
  - Type: HTTP Request  
  - Role: Calls Gemini image generation API with the prepared prompt.  
  - Input: Image prompt  
  - Output: Image file or URL  

- **Edit Fields1**  
  - Type: Set  
  - Role: Modifies data fields, possibly setting image metadata or paths.  

- **Convert to File**  
  - Type: Convert to File  
  - Role: Converts image data to a file format suitable for upload.  

- **Resize Image**  
  - Type: Edit Image  
  - Role: Resizes image to WordPress recommended dimensions.  

- **Upload Image To WP**  
  - Type: HTTP Request  
  - Role: Uploads the processed image file to WordPress media library via REST API.  

- **Create a post**  
  - Type: WordPress node  
  - Role: Creates a new blog post with content and metadata.  

- **Update a post**  
  - Type: WordPress node  
  - Role: Updates the newly created post with final data.  

- **Set Featured Image**  
  - Type: HTTP Request  
  - Role: Assigns the uploaded image as the featured image for the WordPress post.

- Edge Cases: API failures, image size issues, upload errors, authentication failures.

---

#### 1.6 Blog Metadata and Publishing

**Overview:**  
Generates blog metadata, performs any final data processing, and manages the publishing lifecycle on WordPress.

**Nodes Involved:**  
- Generate Blog Metadata (LangChain Agent)  
- Code (Code node)  
- Get row(s) in sheet (Google Sheets)  
- Schedule Trigger (Disabled)  

**Node Details:**

- **Generate Blog Metadata**  
  - Type: LangChain Agent  
  - Role: Creates metadata such as tags, categories, SEO descriptions.  
  - Input: Blog content or title  
  - Output: Metadata JSON  

- **Code**  
  - Type: Code  
  - Role: Processes or reformats metadata for WordPress consumption.  

- **Get row(s) in sheet**  
  - Type: Google Sheets  
  - Role: Retrieves data rows, possibly video or blog post references from a Google Sheet.  

- **Schedule Trigger**  
  - Type: Schedule Trigger (disabled)  
  - Role: Could trigger the workflow periodically if enabled.

---

### 3. Summary Table

| Node Name                  | Node Type                            | Functional Role                         | Input Node(s)               | Output Node(s)                | Sticky Note                  |
|----------------------------|------------------------------------|---------------------------------------|-----------------------------|------------------------------|------------------------------|
| Check Youtube Channel       | RSS Feed Read Trigger               | Trigger workflow on new YouTube video  | -                           | Get Transcript               |                              |
| Get Transcript             | HTTP Request                       | Retrieve transcript for video          | Check Youtube Channel         | Clean Data                   |                              |
| Clean Data                 | Code                              | Clean raw transcript text               | Get Transcript               | Search intent and style      |                              |
| Search intent and style    | LangChain Agent                   | Analyze content intent and style        | Clean Data                   | Table of Contents            |                              |
| Structured Output Parser1  | Output Parser                    | Parse AI output to structured format    | Search intent and style      | Table of Contents            |                              |
| Table of Contents          | LangChain Agent                   | Generate blog TOC                       | Structured Output Parser1    | Create the Sections          |                              |
| Structured Output Parser   | Output Parser                    | Parse TOC AI output                     | Table of Contents            | Create the Sections          |                              |
| Create the Sections        | LangChain Agent                   | Generate blog post sections             | Structured Output Parser     | Split Out1                  |                              |
| Split Out1                 | Split Out                        | Split sections for parallel processing  | Create the Sections          | Generate the Content, Merge  |                              |
| Generate the Content       | LangChain Agent                   | Generate detailed content per section  | Split Out1                  | Merge                       |                              |
| Merge                     | Merge                            | Combine split content parts              | Split Out1, Generate Content | Aggregate1                  |                              |
| Aggregate1                 | Aggregate                       | Aggregate all content parts              | Merge                       | EEAT                        |                              |
| EEAT                      | LangChain Agent                   | Enhance content for trustworthiness     | Aggregate1                  | Content Editor              |                              |
| Content Editor            | LangChain Agent                   | Edit and polish content                  | EEAT                        | Title Maker1                |                              |
| Title Maker1              | LangChain Agent                   | Generate blog post title                 | Content Editor              | Generate Blog Metadata      |                              |
| Generate Blog Metadata    | LangChain Agent                   | Generate SEO metadata                    | Title Maker1                | Code                        |                              |
| Code                      | Code                             | Process metadata                         | Generate Blog Metadata      | Generate Image Prompt + Alt Text |                              |
| Generate Image Prompt + Alt Text | LangChain Agent                   | Create image prompts and alt text        | Code                        | Nano 🍌                    |                              |
| Structured Output Parser3  | Output Parser                    | Parse image prompt output                | Generate Image Prompt + Alt Text | Nano 🍌                |                              |
| Nano 🍌                    | HTTP Request                     | Call Gemini image generation             | Structured Output Parser3    | Edit Fields1                |                              |
| Edit Fields1              | Set                              | Adjust image metadata                    | Nano 🍌                    | Convert to File             |                              |
| Convert to File           | Convert to File                  | Convert image data to file               | Edit Fields1                | Resize Image                |                              |
| Resize Image              | Edit Image                      | Resize image to WordPress specs          | Convert to File             | Upload Image To WP          |                              |
| Upload Image To WP        | HTTP Request                    | Upload image to WordPress media library  | Resize Image                | Create a post               |                              |
| Create a post             | WordPress                       | Create WordPress blog post               | Upload Image To WP          | Update a post               |                              |
| Update a post             | WordPress                       | Update WordPress post                    | Create a post               | Set Featured Image          |                              |
| Set Featured Image        | HTTP Request                    | Set uploaded image as featured image     | Update a post               | -                          |                              |
| Get row(s) in sheet       | Google Sheets                   | Read video/blog data from Google Sheets  | Schedule Trigger            | Get Transcript             |                              |
| Schedule Trigger          | Schedule Trigger (disabled)      | Trigger workflow periodically            | -                           | Get row(s) in sheet         | Disabled - manual or external trigger |
| Ollama Chat Model (multiple) | LM Chat Ollama                  | AI inference for various tasks           | Connected to various agents | Connected agents            |                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Node:**  
   - Add `RSS Feed Read Trigger` node.  
   - Configure with the target YouTube channel RSS URL.  
   - Set polling frequency as desired.

2. **Get Transcript:**  
   - Add an `HTTP Request` node.  
   - Configure to call an API or service that provides YouTube video transcripts, passing the video ID or URL from the RSS feed trigger.

3. **Clean Data:**  
   - Add a `Code` node.  
   - Write JavaScript code to clean and format the transcript text (remove timestamps, special characters).

4. **Intent and Style Analysis:**  
   - Add a `LangChain Agent` node named "Search intent and style".  
   - Connect to an `LM Chat Ollama` node configured with an appropriate Ollama model for text analysis.  
   - Set prompt to determine content style and intent from cleaned transcript.

5. **Structured Output Parsing:**  
   - Add `Structured Output Parser1`.  
   - Configure it to parse the AI output from the previous step into a structured JSON.

6. **Generate Table of Contents:**  
   - Add a `LangChain Agent` node "Table of Contents".  
   - Connect it to the parsed intent/style data.  
   - Connect this agent to an `LM Chat Ollama` node for AI inference.

7. **Parse TOC Output:**  
   - Add `Structured Output Parser` node to parse TOC output.

8. **Create Blog Sections:**  
   - Add `LangChain Agent` node "Create the Sections".  
   - Connect to parsed TOC output.  
   - Connect to `LM Chat Ollama` node.

9. **Split and Generate Content:**  
   - Add `Split Out1` node to split sections for parallel processing.  
   - Add `Generate the Content` LangChain Agent node connected to one output of Split Out1.  
   - Connect another output of Split Out1 directly to a `Merge` node.  
   - Connect output of Generate the Content to the Merge node.

10. **Aggregate Content:**  
    - Add `Aggregate1` node connected to Merge.

11. **EEAT Enhancement:**  
    - Add `EEAT` LangChain Agent node connected to Aggregate1.

12. **Content Editing:**  
    - Add `Content Editor` LangChain Agent node connected to EEAT.

13. **Title Generation:**  
    - Add `Title Maker1` LangChain Agent node connected to Content Editor.

14. **Generate Blog Metadata:**  
    - Add `Generate Blog Metadata` LangChain Agent node connected to Title Maker1.

15. **Process Metadata:**  
    - Add a `Code` node connected to Generate Blog Metadata for any necessary metadata transformations.

16. **Generate Image Prompt and Alt Text:**  
    - Add `Generate Image Prompt + Alt Text` LangChain Agent node connected to the Code node.

17. **Parse Image Prompt Output:**  
    - Add `Structured Output Parser3` node connected to the previous node.

18. **Image Generation (Gemini):**  
    - Add an `HTTP Request` node ("Nano 🍌") configured to call Gemini image generation API with the prompt.

19. **Edit Image Metadata:**  
    - Add a `Set` node ("Edit Fields1") connected to the image generation node to set any required fields.

20. **Convert to File:**  
    - Add `Convert to File` node to convert image data for upload.

21. **Resize Image:**  
    - Add `Edit Image` node to resize to WordPress recommended dimensions.

22. **Upload Image to WordPress:**  
    - Add `HTTP Request` node configured to upload the image via WordPress REST API.

23. **Create WordPress Post:**  
    - Add `WordPress` node configured to create a new post, using the generated content and metadata.

24. **Update WordPress Post:**  
    - Add `WordPress` node to update the post with any additional data.

25. **Set Featured Image:**  
    - Add `HTTP Request` node to set the uploaded image as the featured image of the WordPress post.

26. **Optional Google Sheet Integration:**  
    - Add `Google Sheets` node to read video or post data.  
    - Connect to the workflow start if using scheduled runs.

27. **Optional Schedule Trigger:**  
    - Add a `Schedule Trigger` node to automate runs (currently disabled in original).

28. **Connect all nodes according to the flow described in section 2.**

29. **Credentials:**  
    - Set up credentials for Ollama models (API key or local server).  
    - Configure HTTP Requests with appropriate authentication (Gemini API, WordPress REST API with OAuth2 or Application Password).  
    - Set up Google Sheets credentials if used.

---

### 5. General Notes & Resources

| Note Content                                                                                      | Context or Link                                                |
|--------------------------------------------------------------------------------------------------|----------------------------------------------------------------|
| Workflow uses multiple Ollama Chat Model nodes for different AI tasks (intent, content, metadata). | Ollama AI integration in n8n                                   |
| Gemini image generation is invoked via an HTTP Request node named "Nano 🍌".                      | Gemini AI image generation API                                 |
| WordPress nodes require REST API access and appropriate permissions for media and post management.| WordPress REST API documentation: https://developer.wordpress.org/rest-api/ |
| Workflow includes disabled schedule trigger - can be enabled for periodic automation.            | Scheduling in n8n: https://docs.n8n.io/nodes/n8n-nodes-base.scheduleTrigger/ |
| Google Sheets integration is optional for external data reference or control.                    | Google Sheets node docs: https://docs.n8n.io/nodes/n8n-nodes-base.googleSheets/ |

---

**Disclaimer:**  
This document is generated exclusively from an automated n8n workflow. All content respects applicable policies and contains no illegal or protected elements. Data processed is legal and public.