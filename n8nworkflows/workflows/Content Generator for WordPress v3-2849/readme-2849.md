Content Generator for WordPress v3

https://n8nworkflows.xyz/workflows/content-generator-for-wordpress-v3-2849


# Content Generator for WordPress v3

### 1. Workflow Overview

This workflow, **Content Generator for WordPress v3**, automates the entire content creation, optimization, and publishing process for WordPress blogs. It is designed for content creators, marketers, and business owners who want to efficiently generate SEO-friendly, branded blog posts with minimal manual effort.

The workflow integrates multiple AI models (OpenAI GPT-4 and Anthropic Claude), Airtable for content management, WordPress REST API for publishing, RankMath SEO plugin for SEO optimization, and an image generation service for branded featured images. It also includes Slack notifications and updates Airtable with publishing status and URLs.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception and Content Management:** Triggering the workflow and retrieving brand guidelines and keywords from Airtable.
- **1.2 AI Content Generation:** Creating blog post titles, structure, and chapter content using AI agents with structured output parsing.
- **1.3 Content Assembly and Validation:** Merging chapter titles and texts, validating data consistency, and formatting the final article text.
- **1.4 SEO Optimization:** Generating SEO metadata and updating RankMath SEO plugin via HTTP requests.
- **1.5 Featured Image Generation and Upload:** Creating branded images, uploading them to WordPress, and associating them with posts.
- **1.6 Publishing and Status Updates:** Posting the article on WordPress, updating Airtable with post URLs, and sending Slack notifications.
- **1.7 Error Handling:** Managing data inconsistencies and responding to errors gracefully.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and Content Management

**Overview:**  
This block initiates the workflow (manually or scheduled), retrieves brand guidelines and keywords from Airtable, and prepares the content parameters for AI generation.

**Nodes Involved:**  
- When clicking ‘Test workflow’ (Manual Trigger)  
- GET Brand Guidelines (Airtable)  
- GET Keywords (Airtable)  
- Aggregate (Aggregate)  
- If (Conditional check)  
- Post Parameters (Set)  
- Loop Over Items (SplitInBatches)  
- Settings (Set)

**Node Details:**

- **When clicking ‘Test workflow’**  
  - Type: Manual Trigger  
  - Role: Entry point for manual execution  
  - Config: Default manual trigger, no parameters  
  - Outputs: Connects to GET Brand Guidelines  
  - Edge cases: None (manual trigger)

- **GET Brand Guidelines**  
  - Type: Airtable  
  - Role: Fetches brand tone, style, and voice guidelines  
  - Config: Airtable base and table configured for brand guidelines  
  - Outputs: Aggregated data for AI prompt  
  - Edge cases: Airtable API errors, empty data

- **GET Keywords**  
  - Type: Airtable  
  - Role: Retrieves SEO keywords for the post  
  - Config: Airtable base and table for keywords  
  - Outputs: Feeds into conditional check  
  - Edge cases: API errors, missing keywords

- **Aggregate**  
  - Type: Aggregate  
  - Role: Combines multiple Airtable records into a single dataset  
  - Config: Aggregates brand guidelines or keywords  
  - Outputs: Single aggregated item for AI prompt  
  - Edge cases: Empty aggregation

- **If**  
  - Type: Conditional  
  - Role: Checks if keywords exist before proceeding  
  - Config: Condition on keyword presence  
  - Outputs: True path continues, false path halts or error handling  
  - Edge cases: Missing or malformed data

- **Post Parameters**  
  - Type: Set  
  - Role: Sets parameters for post creation including title, keywords, and brand info  
  - Config: Sets variables for downstream nodes  
  - Outputs: Feeds Loop Over Items node  
  - Edge cases: Missing parameters

- **Loop Over Items**  
  - Type: SplitInBatches  
  - Role: Processes each content item individually to avoid overload  
  - Config: Batch size default (usually 1)  
  - Outputs: Feeds AI generation block or error path  
  - Edge cases: Large batch sizes causing timeouts

- **Settings**  
  - Type: Set  
  - Role: Sets AI prompt settings and other configuration variables  
  - Config: Default prompt parameters, model selection, temperature, etc.  
  - Outputs: Feeds AI Agent nodes  
  - Edge cases: Incorrect prompt settings causing poor AI output

---

#### 1.2 AI Content Generation

**Overview:**  
Generates blog post titles, structured outlines, and chapter content using AI agents powered by OpenAI GPT-4 and Anthropic Claude. Uses structured output parsing for consistent data.

**Nodes Involved:**  
- AI Agent - Create Post Title and Structure (Langchain Agent)  
- Create post title and structure (OpenAI)  
- Structured Output Parser (Langchain Output Parser)  
- Check data consistency (If)  
- Create chapters text (OpenAI)  
- Split out chapters (SplitOut)  
- Merge chapters title and text (Merge)  
- Code (Code node for formatting)  
- AI Agent1 (Langchain Agent for Claude)  
- Anthropic Chat Model / Anthropic Chat Model1 (Langchain LM Chat)  
- Wikipedia / Wikipedia1 / Wikipedia2 (Langchain Wikipedia Tool)  
- Calculator / Calculator1 (Langchain Calculator Tool)  
- HTTP Request / HTTP Request1 / HTTP Request2 (Langchain HTTP Request Tool)

**Node Details:**

- **AI Agent - Create Post Title and Structure**  
  - Type: Langchain Agent  
  - Role: Generates post title and structured outline using AI with tools (Wikipedia, Calculator, HTTP Request)  
  - Config: Uses Anthropic Claude or OpenAI as language model, with tool integrations for factual accuracy  
  - Inputs: Brand guidelines, keywords, prompt settings  
  - Outputs: Structured JSON with title and chapter headings  
  - Edge cases: API rate limits, tool failures, malformed output

- **Create post title and structure**  
  - Type: OpenAI Node  
  - Role: Alternative AI generation of title and structure (GPT-4)  
  - Config: Prompt tailored for SEO and brand tone  
  - Inputs: Brand guidelines, keywords  
  - Outputs: Text or structured output  
  - Edge cases: API errors, incomplete responses

- **Structured Output Parser**  
  - Type: Langchain Output Parser  
  - Role: Parses AI output into structured JSON for downstream processing  
  - Config: Schema defined for expected output format  
  - Inputs: AI Agent output  
  - Outputs: Parsed JSON with chapters and titles  
  - Edge cases: Parsing errors if AI output deviates from schema

- **Check data consistency**  
  - Type: If  
  - Role: Validates presence and correctness of AI-generated data before continuing  
  - Config: Checks for required fields in parsed output  
  - Outputs: True path proceeds, false path triggers error response  
  - Edge cases: Missing or malformed data

- **Create chapters text**  
  - Type: OpenAI Node  
  - Role: Generates detailed text for each chapter based on outline  
  - Config: Uses prompts to expand chapter headings into content  
  - Inputs: Chapter titles, brand guidelines  
  - Outputs: Chapter content text  
  - Edge cases: API timeouts, incomplete content

- **Split out chapters**  
  - Type: SplitOut  
  - Role: Splits chapter content into individual items for merging  
  - Config: Splits JSON array of chapters  
  - Outputs: Individual chapter objects  
  - Edge cases: Empty chapters array

- **Merge chapters title and text**  
  - Type: Merge  
  - Role: Combines chapter titles and generated texts into a single dataset  
  - Config: Merges two streams (titles and texts) by index  
  - Outputs: Combined chapter objects  
  - Edge cases: Mismatched array lengths

- **Code**  
  - Type: Code Node  
  - Role: Formats merged chapters into final article text with HTML or markdown  
  - Config: Custom JavaScript code for formatting  
  - Outputs: Final article content string  
  - Edge cases: Code errors, unexpected data structure

- **AI Agent1, Anthropic Chat Model, Wikipedia2, Calculator1, HTTP Request2**  
  - Role: Parallel AI generation path using Anthropic Claude for redundancy or alternative content generation  
  - Config: Similar to AI Agent block but uses Claude model  
  - Edge cases: API limits, inconsistent output

---

#### 1.3 Content Assembly and Validation

**Overview:**  
Finalizes the article text, validates all content, and prepares it for publishing.

**Nodes Involved:**  
- Final article text (Code)  
- Post on Wordpress (WordPress Node)  
- Check data consistency1 (If)  
- Code (Code node for validation)  
- Merge chapters title and text1 (Merge)  
- Final article text1 (Code)  
- Post on Wordpress1 (WordPress Node)

**Node Details:**

- **Final article text / Final article text1**  
  - Type: Code Node  
  - Role: Final formatting of article content for WordPress post body  
  - Config: Combines chapters into a single HTML or markdown string  
  - Outputs: Formatted article content  
  - Edge cases: Formatting errors

- **Post on Wordpress / Post on Wordpress1**  
  - Type: WordPress Node  
  - Role: Creates or updates a post on WordPress via REST API  
  - Config: Uses WordPress credentials, sets post title, content, status, categories  
  - Inputs: Final article text, post parameters  
  - Outputs: WordPress post ID and URL  
  - Edge cases: Authentication errors, API failures, invalid post data

- **Check data consistency1**  
  - Type: If  
  - Role: Validates final data before posting  
  - Outputs: True path continues, false path triggers error handling

- **Merge chapters title and text1**  
  - Type: Merge  
  - Role: Similar to previous merge but for alternative AI generation path

---

#### 1.4 SEO Optimization

**Overview:**  
Generates SEO metadata and updates RankMath SEO plugin on WordPress with focus keywords and meta descriptions.

**Nodes Involved:**  
- SEO Update for RankMath (OpenAI)  
- POST RankMath Info (HTTP Request)  
- SEO Update for RankMath1 (OpenAI)  
- POST RankMath Info1 (HTTP Request)

**Node Details:**

- **SEO Update for RankMath / SEO Update for RankMath1**  
  - Type: OpenAI Node  
  - Role: Generates SEO meta description and focus keywords based on article content  
  - Config: Prompt tailored for SEO metadata generation  
  - Inputs: Final article text, keywords  
  - Outputs: SEO metadata text

- **POST RankMath Info / POST RankMath Info1**  
  - Type: HTTP Request  
  - Role: Sends SEO metadata to RankMath plugin via WordPress REST API  
  - Config: HTTP POST with authentication, targeting RankMath endpoints  
  - Inputs: SEO metadata, post ID  
  - Edge cases: API errors, authentication failures

---

#### 1.5 Featured Image Generation and Upload

**Overview:**  
Generates branded featured images using AI, uploads them to WordPress media library, and associates the image with the post.

**Nodes Involved:**  
- P1 Image Prompt / P1 Image Prompt1 (OpenAI)  
- Aggregate Prompt / Aggregate Prompt1 (Aggregate)  
- Prompt Settings / Prompt Settings1 (Set)  
- Leo - Improve Prompt / Leo - Improve Prompt1 (HTTP Request)  
- Leo - Generate Image / Leo - Generate Image1 (HTTP Request)  
- Wait / Wait1 (Wait)  
- Leo - Get imageId / Leo - Get imageId1 (HTTP Request)  
- Download Image / Download Image1 (HTTP Request)  
- Upload media / Upload media1 (HTTP Request)  
- Set image ID for the post / Set image ID for the post1 (HTTP Request)

**Node Details:**

- **P1 Image Prompt / P1 Image Prompt1**  
  - Type: OpenAI Node  
  - Role: Generates image prompt text for AI image generation service  
  - Config: Uses article content and brand style to create image description  
  - Outputs: Image prompt string

- **Aggregate Prompt / Aggregate Prompt1**  
  - Type: Aggregate  
  - Role: Combines multiple prompt elements into a single prompt string

- **Prompt Settings / Prompt Settings1**  
  - Type: Set  
  - Role: Sets parameters for image generation prompts

- **Leo - Improve Prompt / Leo - Improve Prompt1**  
  - Type: HTTP Request  
  - Role: Sends prompt to external service (Leo) to enhance image prompt quality

- **Leo - Generate Image / Leo - Generate Image1**  
  - Type: HTTP Request  
  - Role: Requests image generation from Leo API

- **Wait / Wait1**  
  - Type: Wait  
  - Role: Pauses workflow to allow image generation to complete

- **Leo - Get imageId / Leo - Get imageId1**  
  - Type: HTTP Request  
  - Role: Retrieves generated image ID from Leo API

- **Download Image / Download Image1**  
  - Type: HTTP Request  
  - Role: Downloads generated image file

- **Upload media / Upload media1**  
  - Type: HTTP Request  
  - Role: Uploads image to WordPress media library

- **Set image ID for the post / Set image ID for the post1**  
  - Type: HTTP Request  
  - Role: Associates uploaded image as featured image for the WordPress post

- **Edge cases:** API timeouts, image generation failures, upload errors, invalid image IDs

---

#### 1.6 Publishing and Status Updates

**Overview:**  
Finalizes publishing by updating Airtable with post URLs and featured image URLs, and sends Slack notifications about the new post.

**Nodes Involved:**  
- POST Blog Info / POST Blog Info1 (Slack)  
- UPDATE Status / UPDATE Status1 / UPDATE Status2 (Airtable)

**Node Details:**

- **POST Blog Info / POST Blog Info1**  
  - Type: Slack  
  - Role: Sends notification to Slack channel with blog post details  
  - Config: Slack webhook or bot token configured  
  - Inputs: Post URL, title, image URL  
  - Edge cases: Slack API errors, network issues

- **UPDATE Status / UPDATE Status1 / UPDATE Status2**  
  - Type: Airtable  
  - Role: Updates Airtable records with published post URL, featured image URL, and status flags  
  - Config: Airtable base and table configured for content tracking  
  - Edge cases: Airtable API errors, record locking

---

#### 1.7 Error Handling

**Overview:**  
Handles errors related to data consistency and AI output validation by responding with error messages or halting the workflow.

**Nodes Involved:**  
- Respond: Error (Respond to Webhook)  
- Check data consistency (If)  
- Check data consistency1 (If)

**Node Details:**

- **Respond: Error**  
  - Type: Respond to Webhook  
  - Role: Sends error response when data validation fails  
  - Config: Returns error message and HTTP status  
  - Edge cases: None (final error response)

- **Check data consistency / Check data consistency1**  
  - Type: If  
  - Role: Validates data and routes to error response if invalid

---

### 3. Summary Table

| Node Name                          | Node Type                         | Functional Role                                  | Input Node(s)                        | Output Node(s)                      | Sticky Note                                                                                   |
|-----------------------------------|----------------------------------|-------------------------------------------------|------------------------------------|-----------------------------------|----------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’      | Manual Trigger                   | Workflow entry point                             | -                                  | GET Brand Guidelines               |                                                                                              |
| GET Brand Guidelines               | Airtable                        | Fetch brand guidelines                           | When clicking ‘Test workflow’       | Aggregate                         |                                                                                              |
| GET Keywords                      | Airtable                        | Fetch SEO keywords                              | Aggregate                         | If                               |                                                                                              |
| Aggregate                        | Aggregate                      | Combine Airtable records                         | GET Brand Guidelines               | GET Keywords                     |                                                                                              |
| If                              | Conditional                    | Check keywords presence                          | GET Keywords                      | Post Parameters / Respond: Error |                                                                                              |
| Post Parameters                 | Set                            | Set parameters for post creation                 | If                               | Loop Over Items                  |                                                                                              |
| Loop Over Items                 | SplitInBatches                 | Process items in batches                          | Post Parameters                  | Settings / (empty batch)          |                                                                                              |
| Settings                       | Set                            | Set AI prompt and model settings                  | Loop Over Items                  | AI Agent - Create Post Title and Structure, Create post title and structure |                                                                                              |
| AI Agent - Create Post Title and Structure | Langchain Agent               | Generate post title and outline                   | Settings                        | Check data consistency           | Uses structured output parsing for consistent AI output                                      |
| Create post title and structure  | OpenAI                         | Alternative AI generation of title and structure | Settings                        | Check data consistency           |                                                                                              |
| Structured Output Parser         | Langchain Output Parser        | Parse AI output into structured JSON              | AI Agent - Create Post Title and Structure | AI Agent - Create Post Title and Structure |                                                                                              |
| Check data consistency           | Conditional                   | Validate AI output data                            | Structured Output Parser, Create post title and structure | Split out chapters / Respond: Error |                                                                                              |
| Split out chapters              | SplitOut                      | Split chapters into individual items              | Check data consistency           | Merge chapters title and text, Create chapters text |                                                                                              |
| Create chapters text            | OpenAI                        | Generate chapter content                           | Split out chapters              | Merge chapters title and text    |                                                                                              |
| Merge chapters title and text  | Merge                         | Combine chapter titles and texts                   | Split out chapters, Create chapters text | Final article text               |                                                                                              |
| Final article text              | Code                          | Format final article content                       | Merge chapters title and text    | Post on Wordpress                |                                                                                              |
| Post on Wordpress              | WordPress                     | Publish post on WordPress                          | Final article text              | SEO Update for RankMath          |                                                                                              |
| SEO Update for RankMath        | OpenAI                        | Generate SEO metadata                              | Post on Wordpress              | POST RankMath Info               |                                                                                              |
| POST RankMath Info             | HTTP Request                  | Update RankMath SEO plugin                         | SEO Update for RankMath        | P1 Image Prompt                 |                                                                                              |
| P1 Image Prompt               | OpenAI                        | Generate image prompt                              | POST RankMath Info             | Aggregate Prompt                |                                                                                              |
| Aggregate Prompt              | Aggregate                    | Combine image prompt elements                       | P1 Image Prompt               | Prompt Settings                |                                                                                              |
| Prompt Settings              | Set                          | Set parameters for image prompt                     | Aggregate Prompt              | Leo - Improve Prompt           |                                                                                              |
| Leo - Improve Prompt         | HTTP Request                  | Enhance image prompt via Leo API                    | Prompt Settings              | Leo - Generate Image           |                                                                                              |
| Leo - Generate Image         | HTTP Request                  | Request image generation                            | Leo - Improve Prompt         | Wait                          |                                                                                              |
| Wait                        | Wait                         | Pause for image generation completion               | Leo - Generate Image         | Leo - Get imageId              |                                                                                              |
| Leo - Get imageId            | HTTP Request                  | Retrieve generated image ID                          | Wait                        | Download Image                |                                                                                              |
| Download Image               | HTTP Request                  | Download generated image                             | Leo - Get imageId            | Upload media                  |                                                                                              |
| Upload media                | HTTP Request                  | Upload image to WordPress media library              | Download Image               | Set image ID for the post     |                                                                                              |
| Set image ID for the post   | HTTP Request                  | Associate image as featured image for post           | Upload media                | POST Blog Info                |                                                                                              |
| POST Blog Info              | Slack                        | Send Slack notification about new post               | Set image ID for the post   | UPDATE Status2                |                                                                                              |
| UPDATE Status2             | Airtable                     | Update Airtable with post URL and status              | POST Blog Info              | UPDATE Status                 |                                                                                              |
| UPDATE Status              | Airtable                     | Update Airtable record status                          | UPDATE Status2              | Loop Over Items              |                                                                                              |
| Respond: Error             | Respond to Webhook           | Send error response on data inconsistency              | Check data consistency (false) | -                            |                                                                                              |

*Note:* The workflow contains a parallel path using Anthropic Claude AI with nodes suffixed by "1" (e.g., AI Agent1, Post on Wordpress1) that mirrors the OpenAI path for redundancy or alternative content generation.

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Type: Manual Trigger  
   - Purpose: Start workflow manually.

2. **Add Airtable Nodes to Fetch Brand Guidelines and Keywords**  
   - Configure Airtable credentials and base/table for brand guidelines.  
   - Configure Airtable node for keywords retrieval.

3. **Add Aggregate Node**  
   - Aggregate brand guidelines records into one item.

4. **Add Conditional (If) Node**  
   - Check if keywords exist; if not, route to error handling.

5. **Add Set Node (Post Parameters)**  
   - Set variables for post creation such as title, keywords, brand voice.

6. **Add SplitInBatches Node (Loop Over Items)**  
   - Process each content item individually.

7. **Add Set Node (Settings)**  
   - Define AI prompt parameters (model, temperature, max tokens).

8. **Add AI Agent Node (Langchain Agent)**  
   - Configure with OpenAI GPT-4 or Anthropic Claude credentials.  
   - Enable tools: Wikipedia, Calculator, HTTP Request.  
   - Set prompt to generate post title and structured outline.

9. **Add OpenAI Node (Create post title and structure)**  
   - Configure for alternative AI generation of title and structure.

10. **Add Structured Output Parser Node**  
    - Define schema for expected AI output (title, chapters).

11. **Add Conditional Node (Check data consistency)**  
    - Validate AI output; route to error response if invalid.

12. **Add SplitOut Node (Split out chapters)**  
    - Split chapters array into individual items.

13. **Add OpenAI Node (Create chapters text)**  
    - Generate detailed text for each chapter.

14. **Add Merge Node (Merge chapters title and text)**  
    - Combine chapter titles and texts by index.

15. **Add Code Node (Final article text)**  
    - Format merged chapters into final article content (HTML/Markdown).

16. **Add WordPress Node (Post on Wordpress)**  
    - Configure WordPress credentials and REST API.  
    - Set post title, content, status, categories.

17. **Add OpenAI Node (SEO Update for RankMath)**  
    - Generate SEO metadata (focus keywords, meta description).

18. **Add HTTP Request Node (POST RankMath Info)**  
    - Send SEO metadata to RankMath plugin via WordPress API.

19. **Add OpenAI Node (P1 Image Prompt)**  
    - Generate image prompt text for featured image.

20. **Add Aggregate Node (Aggregate Prompt)**  
    - Combine prompt elements.

21. **Add Set Node (Prompt Settings)**  
    - Set parameters for image generation.

22. **Add HTTP Request Nodes (Leo - Improve Prompt, Leo - Generate Image)**  
    - Call external Leo API to improve prompt and generate image.

23. **Add Wait Node**  
    - Pause to allow image generation.

24. **Add HTTP Request Node (Leo - Get imageId)**  
    - Retrieve generated image ID.

25. **Add HTTP Request Node (Download Image)**  
    - Download generated image file.

26. **Add HTTP Request Node (Upload media)**  
    - Upload image to WordPress media library.

27. **Add HTTP Request Node (Set image ID for the post)**  
    - Associate uploaded image as featured image for the post.

28. **Add Slack Node (POST Blog Info)**  
    - Send Slack notification with post details.

29. **Add Airtable Nodes (UPDATE Status, UPDATE Status1, UPDATE Status2)**  
    - Update Airtable records with publishing status and URLs.

30. **Add Respond to Webhook Node (Respond: Error)**  
    - Send error response if data validation fails.

31. **Configure Credentials**  
    - OpenAI API key  
    - Anthropic Claude API key  
    - Airtable API key and base/table info  
    - WordPress REST API credentials (OAuth2 or Application Password)  
    - Slack webhook or bot token  
    - Leo API credentials for image generation

32. **Optional: Add Parallel AI Path**  
    - Duplicate AI generation nodes using Anthropic Claude for redundancy.

33. **Test Workflow**  
    - Trigger manually, verify data flow, AI outputs, image generation, and publishing.

---

### 5. General Notes & Resources

| Note Content                                                                                         | Context or Link                                                                                         |
|----------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| This workflow is an updated version of the Ultimate Content Generator for WordPress v2.             | https://n8n.io/workflows/2737-ultimate-content-generator-for-wordpress/                                |
| Includes AI Agent Tool node with structured output for consistent AI responses.                      |                                                                                                       |
| Updates Airtable with published blog post URL and featured image URL for content tracking.          |                                                                                                       |
| Requires adding a small update to WordPress theme’s functions.php for seamless automation.          |                                                                                                       |
| Supports customization of AI prompts, Airtable structure, and integration with other plugins.       |                                                                                                       |
| Recommended to experiment with Airtable views and filters for granular content pipeline control.    |                                                                                                       |
| Extendable to include social media posting or analytics tracking.                                   |                                                                                                       |
| Slack notifications keep team informed of new published content.                                    |                                                                                                       |
| Uses external Leo API for AI-powered branded image generation.                                      |                                                                                                       |

---

This comprehensive documentation enables advanced users and AI agents to understand, reproduce, and modify the Content Generator for WordPress v3 workflow effectively, anticipating integration points and potential failure modes.