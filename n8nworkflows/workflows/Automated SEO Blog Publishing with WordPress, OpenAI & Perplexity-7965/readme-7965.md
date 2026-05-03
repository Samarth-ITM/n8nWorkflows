Automated SEO Blog Publishing with WordPress, OpenAI & Perplexity

https://n8nworkflows.xyz/workflows/automated-seo-blog-publishing-with-wordpress--openai---perplexity-7965


# Automated SEO Blog Publishing with WordPress, OpenAI & Perplexity

### 1. Workflow Overview

This workflow, titled **"Automated SEO Blog Publishing with WordPress, OpenAI & Perplexity"**, automates the entire process of generating SEO-optimized blog content, creating related images, and publishing posts on a WordPress site. It leverages OpenAI language models and Perplexity AI for content planning and generation, Google Sheets as a data source, and integrates image generation and upload workflows to enrich posts visually.

**Target Use Cases:**
- Automated content marketing teams looking to scale blog production.
- SEO specialists aiming to streamline keyword-based article generation.
- Agencies or individuals automating WordPress content updates with AI-generated material.
- Workflows that require integration of AI text generation, image generation, and CMS publishing.

**Logical blocks grouped by functionality and node dependencies:**

- **1.1 Trigger and Data Input:**  
  Scheduling the workflow and retrieving the input data (topics/keywords) from Google Sheets.

- **1.2 Input Preparation and Limiting:**  
  Applying limits to the data set and setting credentials for downstream nodes.

- **1.3 Content Planning:**  
  Using OpenAI and Perplexity AI to plan article topics and structure.

- **1.4 Content Generation:**  
  Generating structured article drafts with OpenAI models and parsing the structured output.

- **1.5 Image Generation and Processing:**  
  Creating AI-generated images related to the articles, uploading them to Cloudinary and WordPress, and associating images with posts.

- **1.6 Post Creation and Publication:**  
  Creating WordPress posts, adding excerpts, and updating Google Sheets with post metadata.

- **1.7 Batch Processing and Aggregation:**  
  Splitting data into batches, looping over items for parallel processing, and aggregating results.

---

### 2. Block-by-Block Analysis

#### 1.1 Trigger and Data Input

- **Overview:**  
  This block schedules the workflow execution and fetches topics or keywords from a Google Sheets database to serve as input for content generation.

- **Nodes Involved:**  
  - Schedule Trigger  
  - Get data databse (Google Sheets)

- **Node Details:**

  - **Schedule Trigger**  
    - Type: Schedule Trigger  
    - Role: Starts the workflow at configured time intervals.  
    - Config: Default scheduling parameters (not specified).  
    - Input: None (trigger).  
    - Output: Triggers next node "Get data databse".  
    - Failure modes: Missed triggers due to server downtime or misconfigured schedule.

  - **Get data databse (Google Sheets)**  
    - Type: Google Sheets  
    - Role: Reads input data (topics/keywords) from a specified Google Sheet.  
    - Configuration: Spreadsheet ID and Sheet name configured (details not provided).  
    - Input: Trigger from Schedule Trigger.  
    - Output: Passes rows to "Limit" node.  
    - Failure modes: API quota exceeded, invalid credentials, connectivity issues.

#### 1.2 Input Preparation and Limiting

- **Overview:**  
  Applies a limit to the number of input items to process and sets necessary credentials and input data for subsequent nodes.

- **Nodes Involved:**  
  - Limit  
  - Set input data/credintenials

- **Node Details:**

  - **Limit**  
    - Type: Limit  
    - Role: Restricts the number of rows processed (e.g., top N keywords).  
    - Configuration: Limit number (not specified).  
    - Input: Rows from Google Sheets.  
    - Output: Passes limited rows to "Set input data/credintenials".  
    - Failure modes: Misconfiguration could result in zero or too many items.

  - **Set input data/credintenials**  
    - Type: Set  
    - Role: Prepares and sets variables/credentials for downstream AI processing nodes.  
    - Configuration: Sets parameters such as API keys or variables needed by AI nodes.  
    - Input: Limited rows.  
    - Output: Passes data to "Content planer".  
    - Failure modes: Missing or invalid credentials, variable misassignment.

#### 1.3 Content Planning

- **Overview:**  
  Uses OpenAI Chat and Perplexity AI tools to generate a content plan based on input data, structuring the topics and generating outlines.

- **Nodes Involved:**  
  - OpenAI Chat Model  
  - Message a model in Perplexity  
  - Content planer  
  - Structured Output Parser1  
  - Split Out

- **Node Details:**

  - **OpenAI Chat Model**  
    - Type: Langchain OpenAI Chat  
    - Role: Generates initial content plan prompts or responses.  
    - Configuration: Model and prompt setup (not detailed).  
    - Input: Data from "Set input data/credintenials".  
    - Output: Feeds "Content planer" node.  
    - Failure modes: API errors, prompt errors, rate limits.

  - **Message a model in Perplexity**  
    - Type: Perplexity Tool  
    - Role: Supplements content plan generation with Perplexity AI responses.  
    - Configuration: Query and API settings.  
    - Input: Feeds into "Content planer".  
    - Output: Adds to content planning data.  
    - Failure modes: API downtime, auth errors.

  - **Content planer**  
    - Type: Langchain Agent  
    - Role: Combines inputs from OpenAI and Perplexity, generating a detailed content plan.  
    - Configuration: Agent setup for planning tasks.  
    - Input: From OpenAI Chat Model and Perplexity Tool.  
    - Output: Passes structured content plan to "Split Out".  
    - Failure modes: Logic errors, incomplete content plans.

  - **Structured Output Parser1**  
    - Type: Langchain Structured Output Parser  
    - Role: Parses AI-generated content plans into structured JSON or usable format.  
    - Input: From OpenAI Chat Model4 (not directly linked here but part of chain).  
    - Output: Provides parsed structured output for "Content planer".  
    - Failure modes: Parsing errors due to malformed AI output.

  - **Split Out**  
    - Type: Split Out  
    - Role: Splits the content plan into individual items for batch processing.  
    - Input: From "Content planer".  
    - Output: Passes items to "Loop Over Items".  
    - Failure modes: Empty splits, handling of unexpected data formats.

#### 1.4 Batch Processing and Aggregation

- **Overview:**  
  Processes each content plan item in batches for efficient handling, loops over each item, and aggregates results after processing.

- **Nodes Involved:**  
  - Loop Over Items  
  - Aggregate

- **Node Details:**

  - **Loop Over Items**  
    - Type: SplitInBatches  
    - Role: Iterates over each content item in batches, enabling parallel or sequential processing.  
    - Configuration: Batch size (not specified).  
    - Input: Split items from "Split Out".  
    - Outputs: Two outputs — one to "Aggregate" for collecting results, another to image generation nodes.  
    - Failure modes: Batch size misconfiguration, partial batch processing errors.

  - **Aggregate**  
    - Type: Aggregate  
    - Role: Collects processed batch items into a single dataset for further use.  
    - Input: From "Loop Over Items".  
    - Output: To "Article wrtiter".  
    - Failure modes: Data loss or mismatch if aggregation misconfigured.

#### 1.5 Content Generation

- **Overview:**  
  Generates the full article content using OpenAI language models, including detailed writing and structuring.

- **Nodes Involved:**  
  - Article wrtiter  
  - OpenAI Chat Model1  
  - OpenAI Chat Model4 (auxiliary)  
  - Structured Output Parser1 (for structured output parsing)

- **Node Details:**

  - **Article wrtiter**  
    - Type: Langchain Chain LLM  
    - Role: Generates article text using chained prompts or steps.  
    - Configuration: Chain of prompts and model parameters tailored for article writing.  
    - Input: Aggregated content plan from "Aggregate".  
    - Output: To "Create a post".  
    - Failure modes: Model errors, prompt misconfiguration, incomplete text generation.

  - **OpenAI Chat Model1**  
    - Type: Langchain OpenAI Chat  
    - Role: Supports Article wrtiter node with chat completions.  
    - Configuration: Model tuning for article generation.  
    - Input: Chained with Article wrtiter.  
    - Output: Feeding the article generation chain.  
    - Failure modes: API limits, prompt issues.

  - **OpenAI Chat Model4**  
    - Type: Langchain OpenAI Chat  
    - Role: Provides structured responses for parsing.  
    - Input: Not directly linked but used to feed "Structured Output Parser1".  
    - Failure modes: Parsing errors if output is malformed.

  - **Structured Output Parser1**  
    - As above in Content Planning.

#### 1.6 Image Generation and Processing

- **Overview:**  
  This block generates images related to articles using AI models, uploads images to Cloudinary and WordPress, and sets images on WordPress posts.

- **Nodes Involved:**  
  - Write query too image AI generator  
  - Generate an image  
  - Generate an image2  
  - Generate an image3 (disabled)  
  - Post image Cloudinary2  
  - Set Output Image  
  - upload image to wordpress1  
  - Set Image on Wordpress Post1  
  - Set Image on Wordpress Post2  
  - Append or update row in sheet1  
  - Append or update row in sheet

- **Node Details:**

  - **Write query too image AI generator**  
    - Type: Langchain Chain LLM  
    - Role: Creates prompts for AI image generation models based on article content.  
    - Input: From "WordPress excerpt add1" and "Article wrtiter".  
    - Output: To image generation nodes.  
    - Failure modes: Incorrect prompt generation, empty prompts.

  - **Generate an image**  
    - Type: Langchain OpenAI (image generation)  
    - Role: Generates images using OpenAI's image models.  
    - Input: From "Write query too image AI generator".  
    - Output: To "upload image to wordpress1".  
    - Failure modes: API errors, generation failures.

  - **Generate an image2**  
    - Type: Langchain OpenAI (image generation)  
    - Role: Secondary image generation for batch items.  
    - Input: From "Loop Over Items".  
    - Output: To "Post image Cloudinary2".  
    - Failure modes: As above.

  - **Generate an image3** (disabled)  
    - Type: Langchain Google Gemini (image generation)  
    - Role: Alternative image generation (disabled).  
    - Failure modes: None (disabled).

  - **Post image Cloudinary2**  
    - Type: HTTP Request  
    - Role: Uploads generated images to Cloudinary cloud storage.  
    - Input: From image generation nodes.  
    - Output: To "Set Output Image".  
    - Failure modes: Network errors, authentication failures.

  - **Set Output Image**  
    - Type: Set  
    - Role: Sets image URL or metadata for downstream processing.  
    - Input: From "Post image Cloudinary2".  
    - Output: Back to "Loop Over Items".  
    - Failure modes: Data mapping errors.

  - **upload image to wordpress1**  
    - Type: HTTP Request  
    - Role: Uploads images to WordPress media library.  
    - Input: From "Generate an image" or "Generate an image3".  
    - Output: To "Set Image on Wordpress Post2".  
    - Failure modes: WordPress API errors, auth issues.

  - **Set Image on Wordpress Post1 / Post2**  
    - Type: HTTP Request  
    - Role: Associates uploaded images with WordPress posts as featured images or inline images.  
    - Input: From upload nodes.  
    - Output: To "Append or update row in sheet" nodes.  
    - Failure modes: API errors, incorrect post IDs.

  - **Append or update row in sheet / sheet1**  
    - Type: Google Sheets  
    - Role: Updates Google Sheets with post metadata including image URLs and post IDs.  
    - Input: From image setting nodes.  
    - Failure modes: API quota, write errors.

#### 1.7 Post Creation and Publication

- **Overview:**  
  Creates WordPress posts with generated content and excerpts, then updates metadata.

- **Nodes Involved:**  
  - Create a post  
  - WordPress excerpt add1

- **Node Details:**

  - **Create a post (WordPress)**  
    - Type: WordPress node  
    - Role: Creates a new WordPress blog post with generated article content.  
    - Configuration: Post type, title, content mapping from article writer output.  
    - Input: From "Article wrtiter".  
    - Output: To "WordPress excerpt add1".  
    - Failure modes: Authentication errors, API limits, malformed content.

  - **WordPress excerpt add1**  
    - Type: HTTP Request  
    - Role: Updates or adds an excerpt to the WordPress post.  
    - Input: From "Create a post".  
    - Output: To image generation query nodes (e.g., "Write query too image AI generator" and "Write query too pexels").  
    - Failure modes: API errors, invalid post references.

---

### 3. Summary Table

| Node Name                   | Node Type                               | Functional Role                             | Input Node(s)                | Output Node(s)                                | Sticky Note |
|-----------------------------|---------------------------------------|---------------------------------------------|-----------------------------|-----------------------------------------------|-------------|
| Schedule Trigger            | Schedule Trigger                      | Starts workflow on schedule                   | None                        | Get data databse                              |             |
| Get data databse            | Google Sheets                        | Reads input keywords/topics                    | Schedule Trigger            | Limit                                         |             |
| Limit                      | Limit                                | Limits number of input rows                    | Get data databse            | Set input data/credintenials                   |             |
| Set input data/credintenials | Set                                  | Prepares variables and credentials             | Limit                       | Content planer                                |             |
| OpenAI Chat Model           | Langchain OpenAI Chat                | AI content planning input                        | Set input data/credintenials | Content planer                                |             |
| Message a model in Perplexity | Perplexity Tool                      | Supplements content plan with Perplexity AI      | Set input data/credintenials | Content planer                                |             |
| Content planer              | Langchain Agent                     | Combines AI inputs to generate content plan     | OpenAI Chat Model, Perplexity | Split Out                                     |             |
| Structured Output Parser1    | Langchain Output Parser Structured  | Parses structured AI output                      | OpenAI Chat Model4          | Content planer                                |             |
| OpenAI Chat Model4          | Langchain OpenAI Chat                | Provides structured AI responses                  | - (chained internally)      | Structured Output Parser1                      |             |
| Split Out                  | Split Out                            | Splits content plan into individual items        | Content planer              | Loop Over Items                               |             |
| Loop Over Items            | SplitInBatches                      | Processes items in batches                       | Split Out                  | Aggregate, Generate an image1, Generate an image2 |             |
| Aggregate                  | Aggregate                           | Collects processed batch items                   | Loop Over Items             | Article wrtiter                              |             |
| Article wrtiter             | Langchain Chain LLM                  | Generates article content                        | Aggregate                   | Create a post                                |             |
| OpenAI Chat Model1          | Langchain OpenAI Chat                | Supports article writing                         | Article wrtiter             | Article wrtiter (chain)                       |             |
| Create a post              | WordPress                           | Creates WordPress post                          | Article wrtiter             | WordPress excerpt add1                        |             |
| WordPress excerpt add1      | HTTP Request                       | Adds excerpt to WordPress post                  | Create a post               | Write query too image AI generator, Write query too pexels |             |
| Write query too image AI generator | Langchain Chain LLM                  | Creates prompts for image generation             | WordPress excerpt add1       | Generate an image, Generate an image3        |             |
| Generate an image           | Langchain OpenAI                    | Generates images via OpenAI                       | Write query too image AI generator | upload image to wordpress1               |             |
| Generate an image1          | Langchain Google Gemini (disabled) | Generates images (disabled)                      | Loop Over Items             | Post image Cloudinary2                        |             |
| Generate an image2          | Langchain OpenAI                    | Generates images for batch items                  | Loop Over Items             | Post image Cloudinary2                        |             |
| Post image Cloudinary2      | HTTP Request                       | Uploads images to Cloudinary                      | Generate an image1, Generate an image2 | Set Output Image                       |             |
| Set Output Image            | Set                                | Sets image URL metadata                          | Post image Cloudinary2      | Loop Over Items                               |             |
| upload image to wordpress1  | HTTP Request                       | Uploads images to WordPress media library         | Generate an image, Generate an image3 | Set Image on Wordpress Post2             |             |
| Set Image on Wordpress Post1 | HTTP Request                       | Associates image with WordPress post               | upload image to wordpress   | Append or update row in sheet                |             |
| Set Image on Wordpress Post2 | HTTP Request                       | Associates image with WordPress post               | upload image to wordpress1  | Append or update row in sheet1               |             |
| Append or update row in sheet | Google Sheets                      | Updates spreadsheet with post metadata             | Set Image on Wordpress Post1 | -                                            |             |
| Append or update row in sheet1 | Google Sheets                      | Updates spreadsheet with post metadata             | Set Image on Wordpress Post2 | -                                            |             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Node:**  
   - Add a **Schedule Trigger** node to start the workflow on defined intervals.

2. **Add Google Sheets Input:**  
   - Create a **Google Sheets** node to read data from your spreadsheet containing topics or keywords.  
   - Configure with spreadsheet ID, sheet name, and necessary OAuth2 credentials.

3. **Add Limit Node:**  
   - Insert a **Limit** node to restrict the number of rows processed per execution (e.g., limit to 5 or 10).

4. **Set Credentials and Input Data:**  
   - Add a **Set** node to define required variables and credentials (e.g., API keys) for AI nodes.

5. **Configure OpenAI Chat Model Node:**  
   - Add **Langchain OpenAI Chat** node.  
   - Configure with your OpenAI API credentials.  
   - Set prompt templates to start content planning.

6. **Add Perplexity Tool Node:**  
   - Insert a **Perplexity Tool** node.  
   - Configure API credentials and query templates to supplement content planning.

7. **Add Langchain Agent:**  
   - Add **Langchain Agent** node named "Content planer".  
   - Connect OpenAI Chat Model and Perplexity Tool nodes as inputs.  
   - Configure the agent to combine inputs and produce a content plan.

8. **Add Structured Output Parser:**  
   - Add **Langchain Structured Output Parser** node to parse the content plan.  
   - Connect the output of an OpenAI Chat Model node (configured for structured output) to this parser.  
   - Connect parser output back into the "Content planer" node.

9. **Add Split Out Node:**  
   - Add **Split Out** node to split the content plan into individual items for batch processing.

10. **Add SplitInBatches Node:**  
    - Add **SplitInBatches** node named "Loop Over Items".  
    - Connect from "Split Out".  
    - Configure batch size according to processing capacity.

11. **Add Aggregate Node:**  
    - Add **Aggregate** node to collect batch processing results.  
    - Connect output of "Loop Over Items" to this node (first output).

12. **Add Article Writer Node:**  
    - Add **Langchain Chain LLM** node named "Article wrtiter".  
    - Connect from "Aggregate".  
    - Configure with chained prompts and OpenAI API credentials.

13. **Add OpenAI Chat Model1 Node:**  
    - Add supportive **Langchain OpenAI Chat** node (OpenAI Chat Model1).  
    - Connect it as AI language model input for "Article wrtiter".

14. **Add WordPress Post Creation Node:**  
    - Add **WordPress** node named "Create a post".  
    - Configure with your WordPress OAuth2 credentials and site URL.  
    - Map generated article content and titles to post fields.  
    - Connect from "Article wrtiter".

15. **Add HTTP Request for Excerpt:**  
    - Add an **HTTP Request** node named "WordPress excerpt add1".  
    - Configure to update post excerpt via WordPress REST API.  
    - Connect from "Create a post".

16. **Add Image Generation Preparation Node:**  
    - Add **Langchain Chain LLM** node "Write query too image AI generator".  
    - Connect from "WordPress excerpt add1".  
    - Configure prompt to create image generation queries.

17. **Add Image Generation Nodes:**  
    - Add one or more **Langchain OpenAI** nodes named "Generate an image" and "Generate an image2".  
    - Connect from "Write query too image AI generator".  
    - Configure with OpenAI API credentials for image generation.

18. **Add Cloudinary Upload Node:**  
    - Add an **HTTP Request** node named "Post image Cloudinary2".  
    - Configure to upload image data to Cloudinary with API credentials.

19. **Add Set Node for Image Data:**  
    - Add a **Set** node named "Set Output Image".  
    - Configure to set image URLs and metadata for further use.

20. **Add HTTP Request Nodes to Upload Images to WordPress:**  
    - Add **HTTP Request** nodes "upload image to wordpress1" and optionally "upload image to wordpress".  
    - Configure with WordPress media upload REST API and credentials.

21. **Add HTTP Request Nodes to Associate Images with Posts:**  
    - Add **HTTP Request** nodes "Set Image on Wordpress Post1" and "Set Image on Wordpress Post2".  
    - Configure to set featured images or attach images to posts.

22. **Add Google Sheets Update Nodes:**  
    - Add **Google Sheets** nodes "Append or update row in sheet" and "Append or update row in sheet1".  
    - Configure to update the spreadsheet with post and image metadata.

23. **Connect Batch Image Generation Outputs:**  
    - From "Loop Over Items", connect batch processing outputs to image generation nodes as needed.

24. **Test the entire workflow end-to-end** to validate data flow, API credentials, and error handling.

---

### 5. General Notes & Resources

| Note Content                                                                                                  | Context or Link                                                                                      |
|--------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| n8n nodes utilize Langchain integration for OpenAI and structured output parsing.                             | Official n8n Langchain nodes documentation: https://docs.n8n.io/integrations/builtin/nodes/        |
| WordPress REST API is used for post creation and media upload; ensure OAuth2 credentials are correctly set up.| WordPress REST API docs: https://developer.wordpress.org/rest-api/                                  |
| Perplexity AI node is employed for enhanced content planning via AI tools beyond OpenAI.                      | Perplexity API info: https://www.perplexity.ai/api                                                      |
| Cloudinary is used as an image hosting service for generated images before uploading to WordPress.            | Cloudinary API docs: https://cloudinary.com/documentation/api_reference                              |
| Disabled nodes indicate optional or alternative steps (e.g., Google Gemini image generation).                  | Can be enabled or customized based on API access or preference.                                     |
| The workflow depends heavily on API rate limits and authentication; ensure valid credentials and quotas.      | Monitor API usage to avoid disruptions.                                                             |

---

This documentation provides a structured and detailed reference for understanding, reproducing, and modifying the "Automated SEO Blog Publishing with WordPress, OpenAI & Perplexity" n8n workflow. It captures each node’s role, configuration context, data flow, and potential failure points to facilitate robust automation development.