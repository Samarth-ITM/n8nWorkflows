Generate Multi-Platform Content with OpenAI, Tavily Research & Supabase Storage

https://n8nworkflows.xyz/workflows/generate-multi-platform-content-with-openai--tavily-research---supabase-storage-7947


# Generate Multi-Platform Content with OpenAI, Tavily Research & Supabase Storage

### 1. Workflow Overview

This workflow automates the generation, enrichment, formatting, storage, and distribution of multi-platform content using AI and external services. It targets content creators, digital marketers, and web application developers seeking a streamlined pipeline to produce professional, SEO-optimized, and platform-tailored content from input titles and images supplied via Google Sheets.

The workflow’s logic is arranged into the following functional blocks:

- **1.1 Input Reception & Triggering:** Receives new content requests and assets (title and image URL) from a Google Sheets document.
- **1.2 Research Aggregation:** Uses Tavily to research and retrieve relevant articles based on the input title, providing enriched topic data.
- **1.3 AI Content Generation:** An AI agent synthesizes multi-platform content (for websites, blogs, landing pages) from researched topics using OpenAI GPT-4.
- **1.4 Content Parsing & Cleaning:** A code node cleans markdown and formatting artifacts from the AI-generated content to ensure consistency.
- **1.5 Image Processing Pipeline:** Downloads images from Google Drive, uploads to NextCloud, and generates publicly accessible URLs.
- **1.6 Content Storage:** Stores the cleaned content and image URLs in a Supabase database for retrieval by web or API clients.
- **1.7 Error Handling:** Provides execution of an error handling workflow if AI content generation encounters issues.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Triggering  
- **Overview:**  
  Listens for new rows added to a Google Sheets document, extracting the post title and associated image URL to initiate the workflow.

- **Nodes Involved:**  
  - Google_Sheets_Trigger

- **Node Details:**  
  - **Google_Sheets_Trigger**  
    - Type: Google Sheets Trigger  
    - Role: Triggers execution on new spreadsheet rows (polls every minute).  
    - Config: Monitors specific Sheet (gid=0) in a defined Google Sheets document; OAuth2 credentials configured.  
    - Inputs: None (trigger)  
    - Outputs: JSON with TITLE and IMAGE_URL fields from the sheet row.  
    - Edge Cases: API rate limits, credential expiry, empty or malformed rows.  
    - Sticky Note Content:  
      "Google Sheets Workflow Trigger: The post title and image URLs are acquired from the Google Sheets document. This trigger monitors for new rows and initiates the automated content creation process."

#### 2.2 Research Aggregation  
- **Overview:**  
  Uses the Tavily research agent to find the top 3 relevant articles for the input title, providing detailed topic titles and content for AI enrichment.

- **Nodes Involved:**  
  - Tavily_Research_Agent

- **Node Details:**  
  - **Tavily_Research_Agent**  
    - Type: Tavily Research API Node  
    - Role: Searches for 3 top relevant articles on the input title with advanced search depth.  
    - Config: Query set dynamically from Google Sheets TITLE field; topic set as "general"; max_results=3; search_depth="advanced".  
    - Inputs: TITLE from Google_Sheets_Trigger  
    - Outputs: Array of 3 articles with titles and content.  
    - Edge Cases: API quota limits, no results, network failures.  
    - Sticky Note Content:  
      "Tavily Research Agent: Tavily takes your title, finds the three most relevant articles, and seamlessly delivers them to the AI agent for smarter, faster insights. Uses advanced search depth for comprehensive research."

#### 2.3 AI Content Generation  
- **Overview:**  
  An advanced AI agent processes the 3 researched articles, synthesizing multi-platform optimized content (website, blog, landing page) with a strict character limit, tone, and style, including category classification.

- **Nodes Involved:**  
  - Multi_Platform_Content_Generator  
  - OpenAI_GPT4_Model  
  - AI_Content_Strategy_Analyzer  
  - Content_Structure_Parser  
  - Error_Handler_Trigger (connected on error)

- **Node Details:**  
  - **Multi_Platform_Content_Generator**  
    - Type: Langchain Agent Node (AI agent)  
    - Role: Core content generator combining research input into structured, platform-tailored content.  
    - Config:  
      - Input text dynamically interpolates main topic and 3 researched topics with content.  
      - System message defines detailed role as multi-platform content strategist, instructions for style, tone, format, and category classification.  
      - Output parser is linked to Content_Structure_Parser node for structured JSON output.  
      - On error, continues to Error_Handler_Trigger to handle failures gracefully.  
    - Inputs: Output from Tavily_Research_Agent, OpenAI GPT-4 model as language model, AI_Content_Strategy_Analyzer as tool.  
    - Outputs: Structured content JSON with titles, category, and platform-specific content blocks.  
    - Edge Cases: AI rate limits, prompt formatting errors, output parsing failures, invalid or incomplete research data.  
    - Version: Requires Langchain support, OpenAI GPT-4 API access.  
    - Sticky Note Content:  
      "Multi-Platform Content Generator: After Tavily gathers the top three articles, the AI agent steps in with a built-in output parser that guarantees consistent, well-structured results. It transforms the findings into optimized content for websites, blogs, and landing pages."

  - **OpenAI_GPT4_Model**  
    - Type: Langchain OpenAI Chat Model  
    - Role: Provides GPT-4.1-mini model for language generation.  
    - Config: Uses "gpt-4.1-mini" model with default options.  
    - Inputs: Prompts from Multi_Platform_Content_Generator.  
    - Outputs: Text completions used by the AI agent.  
    - Credentials: OpenAI API key required.  
    - Edge Cases: API quota, latency, authentication failures.

  - **AI_Content_Strategy_Analyzer**  
    - Type: Langchain Tool Think Node  
    - Role: Provides additional AI-assisted analytical tools to support content strategy generation.  
    - Inputs/Outputs: Linked as tool to Multi_Platform_Content_Generator.  
    - Edge Cases: Same as AI agent nodes.

  - **Content_Structure_Parser**  
    - Type: Langchain Output Parser Structured  
    - Role: Parses AI output into a strict JSON structure for downstream processing.  
    - Inputs: AI agent raw output.  
    - Outputs: Clean JSON with fields like title, category, and platform content blocks.  
    - Edge Cases: Parsing errors if AI output deviates from expected format.

  - **Error_Handler_Trigger**  
    - Type: Execute Workflow Node  
    - Role: On AI generation error, triggers a separate error-handling workflow to log or notify.  
    - Inputs: Connected as onError path from Multi_Platform_Content_Generator.  
    - Outputs: None.  
    - Edge Cases: Error handling workflow availability.

#### 2.4 Content Parsing & Cleaning  
- **Overview:**  
  Cleans the AI-generated content JSON by removing markdown headings, bold markers, and normalizing line breaks to prepare text for storage and display.

- **Nodes Involved:**  
  - Content_Text_Cleaner

- **Node Details:**  
  - **Content_Text_Cleaner**  
    - Type: Code Node (JavaScript)  
    - Role: Text normalization for titles, article body, and categories.  
    - Logic: Removes markdown headings (lines starting with #), bold section headers, excess line breaks, trims trailing spaces.  
    - Inputs: JSON output from Multi_Platform_Content_Generator.  
    - Outputs: Clean JSON with fields `title`, `article`, `category`.  
    - Edge Cases: Unexpected input format, empty fields.  
    - Sticky Note Content:  
      "Content Text Cleaner: Once the AI has generated content, this node cleans and formats the text, removing markdown artifacts and ensuring consistent structure for database storage."

#### 2.5 Image Processing Pipeline  
- **Overview:**  
  Downloads images from Google Drive using URLs from Google Sheets, uploads them to NextCloud storage, then generates publicly accessible URLs for integration in content.

- **Nodes Involved:**  
  - Google_Drive_Image_Downloader  
  - NextCloud_Image_Uploader  
  - NextCloud_Public_URL_Generator

- **Node Details:**  
  - **Google_Drive_Image_Downloader**  
    - Type: Google Drive Node (Download)  
    - Role: Fetches image files from Google Drive based on URL in the Google Sheets trigger.  
    - Config: File ID extracted dynamically from IMAGE_URL field.  
    - Credentials: Google Drive OAuth2.  
    - Inputs: IMAGE_URL from Google_Sheets_Trigger.  
    - Outputs: Binary image data.  
    - Edge Cases: Permission errors, invalid URLs, large file sizes.  
    - Sticky Note Content:  
      "Google Drive Image Downloader: Downloads images directly from Google Drive using the URLs provided in the Google Sheets trigger, preparing them for upload to NextCloud storage."

  - **NextCloud_Image_Uploader**  
    - Type: NextCloud Node (Upload)  
    - Role: Uploads binary image data to NextCloud storage under /images folder named by TITLE.jpg.  
    - Config: Path dynamically set from TITLE field.  
    - Credentials: NextCloud API credentials required.  
    - Inputs: Binary data from Google_Drive_Image_Downloader; TITLE from Google_Sheets_Trigger.  
    - Outputs: Confirmation of upload.  
    - Edge Cases: Upload failures, path conflicts.  
    - Sticky Note Content:  
      "NextCloud Image Uploader: Uploads downloaded images to NextCloud storage in an organized folder structure, making them accessible for the web application."

  - **NextCloud_Public_URL_Generator**  
    - Type: NextCloud Node (Share URL Generation)  
    - Role: Creates a publicly shareable URL for the uploaded image to be used in content and stored in database.  
    - Config: Shares path /images/TITLE.jpg with shareType=3 (public link).  
    - Inputs: Upload confirmation from NextCloud_Image_Uploader.  
    - Outputs: Public URL string.  
    - Edge Cases: Share permission errors, URL generation failures.  
    - Sticky Note Content:  
      "NextCloud Public URL Generator: Generates public sharing URLs for uploaded images, creating accessible links that can be stored in the database and used by frontend applications."

#### 2.6 Content Storage  
- **Overview:**  
  Stores the finalized, cleaned content along with image public URLs into a Supabase database table named “works,” making it accessible to frontends or APIs.

- **Nodes Involved:**  
  - Supabase_Content_Storage

- **Node Details:**  
  - **Supabase_Content_Storage**  
    - Type: Supabase Node (Insert)  
    - Role: Inserts or updates database entries with `title`, `content` (article), `category`, and `image_url`.  
    - Config: Table `works`; fields mapped dynamically from Content_Text_Cleaner and NextCloud_Public_URL_Generator outputs.  
    - Inputs: Cleaned content JSON and image URL.  
    - Outputs: Database insert confirmation.  
    - Credentials: Supabase API key.  
    - Edge Cases: Database connection errors, schema mismatch.  
    - Sticky Note Content:  
      "Supabase Content Storage: Stores the finalized content (title, article, category, and image URL) in a Supabase database table, making it available for retrieval by web applications and APIs."

---

### 3. Summary Table

| Node Name                     | Node Type                           | Functional Role                         | Input Node(s)                   | Output Node(s)                   | Sticky Note                                                                                          |
|-------------------------------|-----------------------------------|---------------------------------------|---------------------------------|----------------------------------|----------------------------------------------------------------------------------------------------|
| Google_Sheets_Trigger          | Google Sheets Trigger              | Input reception and workflow trigger | None                            | Tavily_Research_Agent            | Google Sheets Workflow Trigger: The post title and image URLs are acquired from the Google Sheets document. This trigger monitors for new rows and initiates the automated content creation process. |
| Tavily_Research_Agent          | Tavily Research API Node           | Research aggregation                   | Google_Sheets_Trigger           | Multi_Platform_Content_Generator | Tavily Research Agent: Tavily takes your title, finds the three most relevant articles, and seamlessly delivers them to the AI agent for smarter, faster insights. Uses advanced search depth for comprehensive research. |
| Multi_Platform_Content_Generator| Langchain Agent Node              | AI multi-platform content generation  | Tavily_Research_Agent, OpenAI_GPT4_Model, AI_Content_Strategy_Analyzer, Content_Structure_Parser (parser) | Content_Text_Cleaner, Error_Handler_Trigger | Multi-Platform Content Generator: After Tavily gathers the top three articles, the AI agent steps in with a built-in output parser that guarantees consistent, well-structured results. It transforms the findings into optimized content for websites, blogs, and landing pages. |
| OpenAI_GPT4_Model              | Langchain OpenAI Chat Model       | Provides GPT-4 language model          | Multi_Platform_Content_Generator | Multi_Platform_Content_Generator |                                                                                                    |
| AI_Content_Strategy_Analyzer   | Langchain Tool Think Node          | AI analytical tool supporting content | Multi_Platform_Content_Generator | Multi_Platform_Content_Generator |                                                                                                    |
| Content_Structure_Parser       | Langchain Output Parser Structured | Parses AI output into JSON structure  | Multi_Platform_Content_Generator | Multi_Platform_Content_Generator |                                                                                                    |
| Content_Text_Cleaner           | Code Node (JavaScript)             | Cleans and normalizes AI-generated text | Multi_Platform_Content_Generator | Google_Drive_Image_Downloader     | Content Text Cleaner: Once the AI has generated content, this node cleans and formats the text, removing markdown artifacts and ensuring consistent structure for database storage. |
| Google_Drive_Image_Downloader  | Google Drive Node (Download)       | Downloads images from Google Drive     | Content_Text_Cleaner, Google_Sheets_Trigger (for image URL) | NextCloud_Image_Uploader         | Google Drive Image Downloader: Downloads images directly from Google Drive using the URLs provided in the Google Sheets trigger, preparing them for upload to NextCloud storage. |
| NextCloud_Image_Uploader       | NextCloud Node (Upload)            | Uploads images to NextCloud storage    | Google_Drive_Image_Downloader   | NextCloud_Public_URL_Generator    | NextCloud Image Uploader: Uploads downloaded images to NextCloud storage in an organized folder structure, making them accessible for the web application. |
| NextCloud_Public_URL_Generator | NextCloud Node (Share URL)         | Generates public shareable URL for images | NextCloud_Image_Uploader        | Supabase_Content_Storage          | NextCloud Public URL Generator: Generates public sharing URLs for uploaded images, creating accessible links that can be stored in the database and used by frontend applications. |
| Supabase_Content_Storage       | Supabase Node (Insert)             | Stores finalized content and image URL | NextCloud_Public_URL_Generator, Content_Text_Cleaner | None                            | Supabase Content Storage: Stores the finalized content (title, article, category, and image URL) in a Supabase database table, making it available for retrieval by web applications and APIs. |
| Error_Handler_Trigger          | Execute Workflow Node              | Handles errors triggered by AI generation | Multi_Platform_Content_Generator (onError) | None                            |                                                                                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Google Sheets Trigger Node:**  
   - Type: Google Sheets Trigger  
   - Configure OAuth2 credentials for Google Sheets.  
   - Set to trigger on new rows in your target sheet (specify document ID and sheet gid=0).  
   - Poll interval: every minute.  
   - Output fields: ensure TITLE and IMAGE_URL columns are included.

2. **Add Tavily Research Agent Node:**  
   - Type: Tavily Research API Node  
   - Connect input from Google_Sheets_Trigger.  
   - Configure Tavily API credentials.  
   - Set query parameter to `={{ $json.TITLE }}`.  
   - Options: topic "general", max_results 3, search_depth "advanced".

3. **Add OpenAI GPT-4 Model Node:**  
   - Type: Langchain OpenAI Chat Model  
   - Set model to "gpt-4.1-mini".  
   - Configure OpenAI credentials with your API key.

4. **Add AI Content Strategy Analyzer Node:**  
   - Type: Langchain Tool Think Node  
   - No additional configuration needed; used as tool by AI agent.

5. **Add Multi-Platform Content Generator Node:**  
   - Type: Langchain Agent Node  
   - Connect input from Tavily_Research_Agent (main), OpenAI_GPT4_Model (ai_languageModel), and AI_Content_Strategy_Analyzer (ai_tool).  
   - Configure prompt: dynamic text integrating main topic and 3 researched topics with content.  
   - System message: detailed role and instructions for multi-platform content creation (as per overview).  
   - Enable output parser linked to Content_Structure_Parser node.  
   - Set onError path to Error_Handler_Trigger node.

6. **Add Content Structure Parser Node:**  
   - Type: Langchain Output Parser Structured  
   - Define JSON schema reflecting expected AI output (title, category, content blocks).  
   - Connect input from Multi_Platform_Content_Generator output parser.

7. **Add Content Text Cleaner Node:**  
   - Type: Code Node (JavaScript)  
   - Paste cleaning script to remove markdown, bold markers, normalize line breaks, and trim spaces.  
   - Connect main output from Multi_Platform_Content_Generator.

8. **Add Google Drive Image Downloader Node:**  
   - Type: Google Drive Node (Download)  
   - Configure OAuth2 credentials for Google Drive.  
   - Set fileId parameter dynamically from `={{ $('Google_Sheets_Trigger').item.json.IMAGE_URL }}`.  
   - Connect input from Content_Text_Cleaner for binary data preparation.

9. **Add NextCloud Image Uploader Node:**  
   - Type: NextCloud Node (Upload)  
   - Configure NextCloud API credentials.  
   - Set upload path: `/images/{{ $('Google_Sheets_Trigger').item.json.TITLE }}.jpg`.  
   - Enable binary data upload.  
   - Connect input from Google_Drive_Image_Downloader output.

10. **Add NextCloud Public URL Generator Node:**  
    - Type: NextCloud Node (Share URL)  
    - Use same path as uploader.  
    - Share type: 3 (public link).  
    - Connect input from NextCloud_Image_Uploader.

11. **Add Supabase Content Storage Node:**  
    - Type: Supabase Node (Insert)  
    - Configure Supabase API credentials.  
    - Target table: `works`.  
    - Map fields:  
      - title = `={{ $('Content_Text_Cleaner').item.json.title }}`  
      - content = `={{ $('Content_Text_Cleaner').item.json.article }}`  
      - category = `={{ $('Content_Text_Cleaner').item.json.category }}`  
      - image_url = `={{ $json.url }}/preview` (from NextCloud_Public_URL_Generator)  
    - Connect input from NextCloud_Public_URL_Generator.

12. **Add Error Handler Trigger Node:**  
    - Type: Execute Workflow Node  
    - Configure to call an error handling workflow designed to log or notify on AI generation errors.  
    - Connect onError output of Multi_Platform_Content_Generator.

13. **Add Sticky Notes:**  
    - Add sticky notes in the workflow editor to document each major block, copying content from the original workflow for clarity.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                   | Context or Link                                                                                      |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| This workflow integrates multiple APIs and services: Google Sheets (input trigger), Tavily (research), OpenAI GPT-4 (content generation), Google Drive (image source), NextCloud (image storage & sharing), and Supabase (database storage). Ensure all credentials are valid and API quotas are sufficient.                                                              | Overall workflow integration                                                                          |
| The multi-platform content generation uses a detailed system prompt and strict output parsing to guarantee platform-specific content quality and format; modifying prompt instructions requires careful testing to maintain output consistency.                                                                                                                 | AI prompt design and output parsing                                                                  |
| Image paths in NextCloud and public URLs must be consistent with naming conventions to avoid conflicts; check that image access permissions allow public sharing.                                                                                                                                                                                               | Image storage and sharing best practices                                                             |
| The workflow assumes text content lengths capped at 1900 characters per platform to comply with output requirements; exceeding length may cause truncation or API errors.                                                                                                                                                                                        | Content length constraints                                                                             |
| For troubleshooting AI generation failures, the workflow routes errors to a dedicated error handler workflow — customize this to your organization's alerting or logging standards.                                                                                                                                                                            | Error handling and monitoring                                                                          |
| [n8n Documentation](https://docs.n8n.io/) provides detailed node configuration references and credential setup guides.                                                                                                                                                                                                                                         | n8n official documentation                                                                            |
| [OpenAI API Docs](https://platform.openai.com/docs/) for managing API keys, models, and usage policies.                                                                                                                                                                                                                                                       | OpenAI integration guide                                                                              |
| [Supabase Docs](https://supabase.com/docs/) for database table setup and API management.                                                                                                                                                                                                                                                                      | Supabase integration guide                                                                            |

---

**Disclaimer:** The provided textual content is derived exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to applicable content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly accessible.