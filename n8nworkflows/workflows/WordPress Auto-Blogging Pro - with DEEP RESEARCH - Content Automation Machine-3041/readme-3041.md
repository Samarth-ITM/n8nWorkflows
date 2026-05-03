WordPress Auto-Blogging Pro - with DEEP RESEARCH - Content Automation Machine

https://n8nworkflows.xyz/workflows/wordpress-auto-blogging-pro---with-deep-research---content-automation-machine-3041


# WordPress Auto-Blogging Pro - with DEEP RESEARCH - Content Automation Machine

### 1. Workflow Overview

This workflow, **WordPress Auto-Blogging Pro - with DEEP RESEARCH - Content Automation Machine**, is a sophisticated content automation system designed for professional bloggers and SEO specialists who want to generate high-quality, SEO-optimized blog posts automatically. It integrates deep, real-time online research with AI-powered content creation, image generation, internal linking, and automated publishing to WordPress, while also backing up all generated content to Google Drive.

The workflow is logically divided into the following functional blocks:

- **1.1 Input Reception & Validation:** Triggered by new rows in Google Sheets, this block receives user inputs such as topic, style, audience, and other parameters, then validates them.
- **1.2 Initial Research & Article Planning:** Conducts preliminary research on the main topic and plans the article structure, including subtopics and chapters.
- **1.3 Deep Research & Chapter Writing Loop:** For each subtopic, performs in-depth research and writes detailed chapters using AI, looping through all chapters.
- **1.4 Image Generation & Processing:** Generates unique images for each chapter and a featured image for the article, resizes and uploads them to WordPress and Google Drive.
- **1.5 Internal Linking:** Retrieves internal website links from the sitemap, limits and aggregates them, then inserts them strategically into chapters and the article.
- **1.6 Final Assembly & Publishing:** Combines all chapters, images, and links into a complete article, converts Markdown to HTML, and publishes it to WordPress (draft or live).
- **1.7 Backup & Documentation:** Saves the entire article and images to a Google Drive folder named after the post title and creates a Google Doc for backup and review.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Validation

- **Overview:**  
  This block triggers the workflow when a new row is added to a Google Sheet. It extracts user inputs such as topic, style, audience, word count, number of chapters, CTAs, and company info. It validates these inputs to ensure the workflow proceeds only with complete and correct data.

- **Nodes Involved:**  
  - Google Sheets Trigger  
  - Limit to last post  
  - Settings (Set node)  
  - Check inputs (If node)  
  - Get post sitemap (HTTP Request)  
  - Get XML file (XML parser)  
  - Split out links (SplitOut)  
  - Limit internal links (Limit)  
  - Aggregate internal links (Aggregate)  
  - Get output (Set)  
  - Create Drive folder (Google Drive)

- **Node Details:**

  - **Google Sheets Trigger**  
    - Type: Trigger node  
    - Role: Initiates workflow on new row addition  
    - Configuration: Monitors a specified Google Sheet and worksheet  
    - Inputs: None (trigger)  
    - Outputs: Row data with all user parameters  
    - Edge cases: Trigger reliability depends on Google Sheets API limits and network stability.

  - **Limit to last post**  
    - Type: Limit node  
    - Role: Ensures only the latest row triggers processing to avoid duplicates  
    - Inputs: Google Sheets Trigger output  
    - Outputs: Single latest row  
    - Edge cases: May skip rows if multiple added quickly.

  - **Settings (Set)**  
    - Type: Set node  
    - Role: Prepares and normalizes input parameters for downstream nodes  
    - Inputs: Limited row data  
    - Outputs: Structured parameters  
    - Edge cases: Expression errors if input fields missing.

  - **Check inputs (If)**  
    - Type: If node  
    - Role: Validates required inputs are present and correctly formatted  
    - Inputs: Settings output  
    - Outputs: True (valid) or False (invalid) path  
    - Edge cases: Stops workflow if inputs invalid.

  - **Get post sitemap (HTTP Request)**  
    - Type: HTTP Request  
    - Role: Retrieves website sitemap XML for internal link extraction  
    - Inputs: None (triggered after input validation)  
    - Outputs: Sitemap XML content  
    - Edge cases: HTTP errors, sitemap unavailability.

  - **Get XML file (XML parser)**  
    - Type: XML parser  
    - Role: Parses sitemap XML to extract URLs  
    - Inputs: HTTP Request output  
    - Outputs: List of URLs  
    - Edge cases: Malformed XML.

  - **Split out links (SplitOut)**  
    - Type: SplitOut node  
    - Role: Splits list of URLs into individual items for processing  
    - Inputs: Parsed URLs  
    - Outputs: Individual URL items  
    - Edge cases: Empty sitemap.

  - **Limit internal links (Limit)**  
    - Type: Limit node  
    - Role: Restricts number of internal links to 20 (default) for SEO optimization  
    - Inputs: Individual URLs  
    - Outputs: Limited URLs  
    - Edge cases: Less than 20 links available.

  - **Aggregate internal links (Aggregate)**  
    - Type: Aggregate node  
    - Role: Reassembles limited URLs into a list for insertion  
    - Inputs: Limited URLs  
    - Outputs: Aggregated list  
    - Edge cases: Empty list.

  - **Get output (Set)**  
    - Type: Set node  
    - Role: Prepares data for next block, including internal links and input parameters  
    - Inputs: Aggregated links and validated inputs  
    - Outputs: Structured data for research and writing  
    - Edge cases: Data mismatch.

  - **Create Drive folder (Google Drive)**  
    - Type: Google Drive node  
    - Role: Creates a new folder named after the blog post title for backups  
    - Inputs: Post title from Get output node  
    - Outputs: Folder ID and metadata  
    - Edge cases: Permission errors, folder name conflicts.

---

#### 2.2 Initial Research & Article Planning

- **Overview:**  
  This block performs initial AI-driven research on the main topic to gather insights and plans the article structure, including subtopics and chapters.

- **Nodes Involved:**  
  - Initial Research (LangChain Agent)  
  - OpenAI Chat Model (for Initial Research)  
  - Structured Output Parser4  
  - Blog Planner (Chain LLM)  
  - OpenAI Chat Model1 (for Blog Planner)  
  - Structured Output Parser1  
  - Check empty output (If)  
  - Sticky Notes (for documentation)

- **Node Details:**

  - **Initial Research (LangChain Agent)**  
    - Type: LangChain Agent node  
    - Role: Performs deep research on the main topic using PerplexityAI or similar tools  
    - Inputs: Structured data from Get output node  
    - Outputs: Research summary and subtopics  
    - Edge cases: API rate limits, incomplete research data.

  - **OpenAI Chat Model (Initial Research)**  
    - Type: OpenAI Chat model node  
    - Role: Supports Initial Research with AI language model for content generation  
    - Inputs: Research prompts  
    - Outputs: Textual research results  
    - Edge cases: Timeout, API errors.

  - **Structured Output Parser4**  
    - Type: Output parser  
    - Role: Parses AI output into structured JSON for subtopics and overview  
    - Inputs: AI text output  
    - Outputs: Parsed structured data  
    - Edge cases: Parsing failures if output format unexpected.

  - **Blog Planner (Chain LLM)**  
    - Type: Chain LLM node  
    - Role: Plans detailed article structure based on research, including chapter titles and outlines  
    - Inputs: Parsed research data  
    - Outputs: Article plan with chapters  
    - Edge cases: Incomplete or ambiguous plans.

  - **OpenAI Chat Model1 (Blog Planner)**  
    - Type: OpenAI Chat model  
    - Role: Supports Blog Planner with AI-generated structure  
    - Inputs: Planning prompts  
    - Outputs: Structured article plan  
    - Edge cases: API errors.

  - **Structured Output Parser1**  
    - Type: Output parser  
    - Role: Parses Blog Planner output into usable JSON structure  
    - Inputs: Blog Planner output  
    - Outputs: Parsed article plan  
    - Edge cases: Parsing errors.

  - **Check empty output (If)**  
    - Type: If node  
    - Role: Validates that Blog Planner output is not empty before proceeding  
    - Inputs: Parsed article plan  
    - Outputs: True or False path  
    - Edge cases: Stops workflow if no plan generated.

---

#### 2.3 Deep Research & Chapter Writing Loop

- **Overview:**  
  For each chapter/subtopic, this block performs detailed research and writes the chapter content using AI. It loops through all chapters, ensuring each is researched and written thoroughly.

- **Nodes Involved:**  
  - Split out chapters (SplitOut)  
  - Loop Over Items2 (SplitInBatches)  
  - Get title chapter and content (Set)  
  - Researcher (LangChain Agent)  
  - OpenAI Chat Model2 (Researcher)  
  - Structured Output Parser3  
  - Copywriter (Chain LLM)  
  - OpenAI Chat Model3 (Copywriter)  
  - Wait2 (Wait node, disabled)  
  - Merge (Merge node)  
  - Sticky Notes (for documentation)

- **Node Details:**

  - **Split out chapters (SplitOut)**  
    - Type: SplitOut node  
    - Role: Splits article plan into individual chapters for processing  
    - Inputs: Parsed article plan  
    - Outputs: Individual chapter data  
    - Edge cases: Empty chapter list.

  - **Loop Over Items2 (SplitInBatches)**  
    - Type: SplitInBatches node  
    - Role: Loops over each chapter for sequential processing to avoid API rate limits  
    - Inputs: Individual chapters  
    - Outputs: Single chapter per iteration  
    - Edge cases: Batch size misconfiguration.

  - **Get title chapter and content (Set)**  
    - Type: Set node  
    - Role: Extracts chapter title and content placeholders for AI processing  
    - Inputs: Current chapter data  
    - Outputs: Structured chapter info  
    - Edge cases: Missing chapter data.

  - **Researcher (LangChain Agent)**  
    - Type: LangChain Agent node  
    - Role: Performs deep research on the current chapter topic using PerplexityAI or similar  
    - Inputs: Chapter title and context  
    - Outputs: Research content for chapter  
    - Edge cases: API limits, incomplete data.

  - **OpenAI Chat Model2 (Researcher)**  
    - Type: OpenAI Chat model  
    - Role: Supports Researcher node with AI-generated research text  
    - Inputs: Research prompts  
    - Outputs: Research text  
    - Edge cases: API errors.

  - **Structured Output Parser3**  
    - Type: Output parser  
    - Role: Parses research output into structured JSON for chapter writing  
    - Inputs: Research text  
    - Outputs: Parsed research data  
    - Edge cases: Parsing failures.

  - **Copywriter (Chain LLM)**  
    - Type: Chain LLM node  
    - Role: Writes the chapter content based on research and chapter title  
    - Inputs: Parsed research data and chapter title  
    - Outputs: Chapter text in Markdown  
    - Edge cases: Incoherent output, API errors.

  - **OpenAI Chat Model3 (Copywriter)**  
    - Type: OpenAI Chat model  
    - Role: Supports Copywriter node with AI-generated text  
    - Inputs: Writing prompts  
    - Outputs: Chapter content  
    - Edge cases: Timeout, rate limits.

  - **Wait2 (Wait node, disabled)**  
    - Type: Wait node  
    - Role: Intended to pause between API calls to avoid rate limits (currently disabled)  
    - Inputs: Copywriter output  
    - Outputs: Delayed output  
    - Edge cases: Disabled, so no effect.

  - **Merge (Merge node)**  
    - Type: Merge node  
    - Role: Combines all chapter outputs into a single collection for final assembly  
    - Inputs: Chapter texts and images  
    - Outputs: Aggregated chapters  
    - Edge cases: Merge conflicts.

---

#### 2.4 Image Generation & Processing

- **Overview:**  
  This block generates unique images for each chapter and a featured image for the article, resizes them, uploads them to WordPress and Google Drive, and updates metadata.

- **Nodes Involved:**  
  - Generate chapter image (OpenAI LangChain)  
  - Upload chapter images to Drive (Google Drive)  
  - Resize Image (Edit Image)  
  - Update image meta data (HTTP Request)  
  - Generate featured image (OpenAI LangChain)  
  - Upload featured image to Drive (Google Drive)  
  - Resize featured image (Edit Image)  
  - Upload featured image (HTTP Request)  
  - Update featured image meta data (HTTP Request)  
  - Wait featured image (Merge)  
  - Sticky Notes (for documentation)

- **Node Details:**

  - **Generate chapter image**  
    - Type: OpenAI LangChain node  
    - Role: Generates an AI image for the current chapter based on prompts  
    - Inputs: Chapter title and context  
    - Outputs: Image data or URL  
    - Edge cases: Image generation failures, low quality.

  - **Upload chapter images to Drive**  
    - Type: Google Drive node  
    - Role: Uploads generated chapter images to the Google Drive folder created earlier  
    - Inputs: Image files  
    - Outputs: Drive file metadata  
    - Edge cases: Permission errors, upload failures.

  - **Resize Image**  
    - Type: Edit Image node  
    - Role: Resizes chapter images to optimal dimensions for web use  
    - Inputs: Uploaded images  
    - Outputs: Resized images  
    - Edge cases: Image corruption.

  - **Update image meta data**  
    - Type: HTTP Request  
    - Role: Updates WordPress media metadata for chapter images  
    - Inputs: Image URLs and metadata  
    - Outputs: Confirmation response  
    - Edge cases: API errors, auth failures.

  - **Generate featured image**  
    - Type: OpenAI LangChain node  
    - Role: Generates a unique featured image for the entire article  
    - Inputs: Article title and summary  
    - Outputs: Featured image data  
    - Edge cases: Same as chapter images.

  - **Upload featured image to Drive**  
    - Type: Google Drive node  
    - Role: Uploads featured image to Google Drive folder  
    - Inputs: Featured image file  
    - Outputs: Drive metadata  
    - Edge cases: Upload errors.

  - **Resize featured image**  
    - Type: Edit Image node  
    - Role: Resizes featured image to WordPress recommended dimensions  
    - Inputs: Uploaded featured image  
    - Outputs: Resized image  
    - Edge cases: Image processing errors.

  - **Upload featured image**  
    - Type: HTTP Request  
    - Role: Uploads resized featured image to WordPress media library  
    - Inputs: Image file  
    - Outputs: WordPress media ID  
    - Edge cases: API errors.

  - **Update featured image meta data**  
    - Type: HTTP Request  
    - Role: Updates metadata for the featured image in WordPress  
    - Inputs: Media ID and metadata  
    - Outputs: Confirmation  
    - Edge cases: Auth failures.

  - **Wait featured image (Merge)**  
    - Type: Merge node  
    - Role: Synchronizes featured image upload with article publishing  
    - Inputs: Image upload outputs  
    - Outputs: Combined data for publishing  
    - Edge cases: Merge conflicts.

---

#### 2.5 Internal Linking

- **Overview:**  
  This block manages internal website links by extracting them from the sitemap, limiting their number, and preparing them for insertion into the article and chapters to improve SEO and navigation.

- **Nodes Involved:**  
  - Get post sitemap (HTTP Request) [also part of Input block]  
  - Get XML file (XML parser) [also part of Input block]  
  - Split out links (SplitOut) [also part of Input block]  
  - Limit internal links (Limit) [also part of Input block]  
  - Aggregate internal links (Aggregate) [also part of Input block]

- **Node Details:**  
  (See details in 2.1 Input Reception & Validation block as these nodes are shared.)

---

#### 2.6 Final Assembly & Publishing

- **Overview:**  
  This block assembles all chapter texts, images, and internal links into a complete article, converts Markdown to HTML, publishes the post to WordPress (optionally as draft), sets featured image and excerpt, and triggers backup.

- **Nodes Involved:**  
  - Merge chapters title and text (Merge)  
  - Combine into article (Code)  
  - Final article in Markdown (Set)  
  - Markdown to HTML (Markdown)  
  - FInal article in HTML (Set)  
  - Post on Wordpress (WordPress node)  
  - Set featured image for post (HTTP Request)  
  - Set excerpt (HTTP Request)  
  - Edit Fields (Set)  
  - Create Doc (Google Docs)  
  - Save texts to Doc (Google Docs)  
  - Sticky Notes (for documentation)

- **Node Details:**

  - **Merge chapters title and text**  
    - Type: Merge node  
    - Role: Combines individual chapter titles and texts into a single dataset  
    - Inputs: Chapter texts from loop  
    - Outputs: Combined chapters  
    - Edge cases: Merge conflicts.

  - **Combine into article (Code)**  
    - Type: Code node (JavaScript)  
    - Role: Programmatically assembles chapters, inserts internal links, and formats the article in Markdown  
    - Inputs: Merged chapters and internal links  
    - Outputs: Complete article Markdown text  
    - Edge cases: Code errors, malformed Markdown.

  - **Final article in Markdown (Set)**  
    - Type: Set node  
    - Role: Prepares final Markdown article for conversion  
    - Inputs: Combined article text  
    - Outputs: Markdown content  
    - Edge cases: Data loss.

  - **Markdown to HTML (Markdown)**  
    - Type: Markdown node  
    - Role: Converts Markdown article to HTML for WordPress publishing  
    - Inputs: Markdown text  
    - Outputs: HTML content  
    - Edge cases: Conversion errors.

  - **FInal article in HTML (Set)**  
    - Type: Set node  
    - Role: Prepares HTML content for WordPress  
    - Inputs: Converted HTML  
    - Outputs: HTML post content  
    - Edge cases: Data mismatch.

  - **Post on Wordpress (WordPress node)**  
    - Type: WordPress node  
    - Role: Publishes the article to WordPress site, either as draft or live post depending on settings  
    - Inputs: HTML content, metadata, featured image ID  
    - Outputs: Post ID and metadata  
    - Edge cases: Authentication errors, API limits.

  - **Set featured image for post (HTTP Request)**  
    - Type: HTTP Request  
    - Role: Associates the uploaded featured image with the WordPress post  
    - Inputs: Post ID and media ID  
    - Outputs: Confirmation  
    - Edge cases: API errors.

  - **Set excerpt (HTTP Request)**  
    - Type: HTTP Request  
    - Role: Sets the post excerpt in WordPress  
    - Inputs: Post ID and excerpt text  
    - Outputs: Confirmation  
    - Edge cases: API errors.

  - **Edit Fields (Set)**  
    - Type: Set node  
    - Role: Prepares data for Google Docs backup  
    - Inputs: Post content and metadata  
    - Outputs: Structured data for backup  
    - Edge cases: Data mismatch.

  - **Create Doc (Google Docs)**  
    - Type: Google Docs node  
    - Role: Creates a new Google Doc for the article backup  
    - Inputs: Post title and content  
    - Outputs: Document ID  
    - Edge cases: Permission errors.

  - **Save texts to Doc (Google Docs)**  
    - Type: Google Docs node  
    - Role: Saves the article content into the created Google Doc  
    - Inputs: Document ID and content  
    - Outputs: Confirmation  
    - Edge cases: API errors.

---

#### 2.7 Backup & Documentation

- **Overview:**  
  This block ensures all generated content and images are backed up securely to Google Drive and Google Docs, organized in a folder named after the blog post title.

- **Nodes Involved:**  
  - Create Drive folder (Google Drive) [from Input block]  
  - Upload chapter images to Drive (Google Drive)  
  - Upload featured image to Drive (Google Drive)  
  - Create Doc (Google Docs)  
  - Save texts to Doc (Google Docs)

- **Node Details:**  
  (See details in previous blocks; these nodes handle backup and storage.)

---

### 3. Summary Table

| Node Name                   | Node Type                          | Functional Role                          | Input Node(s)                 | Output Node(s)                 | Sticky Note                                                                                  |
|-----------------------------|----------------------------------|----------------------------------------|------------------------------|-------------------------------|----------------------------------------------------------------------------------------------|
| Google Sheets Trigger        | Google Sheets Trigger             | Workflow trigger on new sheet row      | None                         | Limit to last post             |                                                                                              |
| Limit to last post           | Limit                            | Limits to latest row                    | Google Sheets Trigger         | Settings                      |                                                                                              |
| Settings                    | Set                              | Normalizes input parameters             | Limit to last post            | Check inputs                  |                                                                                              |
| Check inputs                | If                               | Validates required inputs               | Settings                     | Get post sitemap, Initial Research |                                                                                              |
| Get post sitemap            | HTTP Request                     | Retrieves sitemap XML                    | Check inputs                 | Get XML file                  |                                                                                              |
| Get XML file                | XML Parser                      | Parses sitemap XML                       | Get post sitemap             | Split out links               |                                                                                              |
| Split out links             | SplitOut                        | Splits URLs                             | Get XML file                 | Limit internal links          |                                                                                              |
| Limit internal links        | Limit                           | Limits internal links to 20             | Split out links              | Aggregate internal links      |                                                                                              |
| Aggregate internal links    | Aggregate                      | Aggregates limited links                 | Limit internal links         | Get output                   |                                                                                              |
| Get output                  | Set                             | Prepares data for research and writing | Aggregate internal links, Check inputs | Create Drive folder, Generate featured image, Split out chapters |                                                                                              |
| Create Drive folder         | Google Drive                    | Creates backup folder                    | Get output                   | Get output                   |                                                                                              |
| Initial Research            | LangChain Agent                 | Performs initial topic research          | Check inputs                 | Blog Planner                 |                                                                                              |
| OpenAI Chat Model           | OpenAI Chat Model               | Supports initial research                | Initial Research             | Structured Output Parser4     |                                                                                              |
| Structured Output Parser4   | Output Parser                  | Parses initial research output           | OpenAI Chat Model            | Blog Planner                 |                                                                                              |
| Blog Planner               | Chain LLM                      | Plans article structure                   | Structured Output Parser4    | Check empty output           |                                                                                              |
| OpenAI Chat Model1          | OpenAI Chat Model               | Supports blog planner                     | Blog Planner                | Structured Output Parser1     |                                                                                              |
| Structured Output Parser1   | Output Parser                  | Parses blog planner output                | OpenAI Chat Model1           | Check empty output           |                                                                                              |
| Check empty output          | If                             | Validates blog planner output             | Structured Output Parser1    | Split out chapters, Blog Planner |                                                                                              |
| Split out chapters          | SplitOut                       | Splits article into chapters              | Check empty output           | Loop Over Items              |                                                                                              |
| Loop Over Items2            | SplitInBatches                 | Loops over chapters for processing        | Split out chapters           | Get title chapter and content, Researcher |                                                                                              |
| Get title chapter and content | Set                           | Extracts chapter title and content        | Loop Over Items2             | Merge chapters title and text |                                                                                              |
| Researcher                 | LangChain Agent                 | Performs deep research per chapter        | Loop Over Items2             | Copywriter                   |                                                                                              |
| OpenAI Chat Model2          | OpenAI Chat Model               | Supports researcher node                   | Researcher                  | Structured Output Parser3     |                                                                                              |
| Structured Output Parser3   | Output Parser                  | Parses deep research output                 | OpenAI Chat Model2           | Copywriter                   |                                                                                              |
| Copywriter                 | Chain LLM                      | Writes chapter content                      | Structured Output Parser3    | Wait2                       |                                                                                              |
| OpenAI Chat Model3          | OpenAI Chat Model               | Supports copywriter node                    | Copywriter                  | Wait2                       |                                                                                              |
| Wait2                      | Wait                          | (Disabled) Intended to avoid rate limits  | Copywriter                  | Loop Over Items2             |                                                                                              |
| Merge                      | Merge                         | Combines chapter texts and images          | Loop Over Items, Aggregate internal links | Merge chapters title and text |                                                                                              |
| Generate chapter image      | OpenAI LangChain               | Generates AI image per chapter              | Loop Over Items              | Upload chapter images to Drive, Resize Image |                                                                                              |
| Upload chapter images to Drive | Google Drive                | Uploads chapter images to Drive             | Generate chapter image       | Update image meta data       |                                                                                              |
| Resize Image               | Edit Image                    | Resizes chapter images                       | Upload chapter images to Drive | Update image meta data       |                                                                                              |
| Update image meta data      | HTTP Request                  | Updates WordPress metadata for chapter images | Resize Image               | Merge1                      |                                                                                              |
| Generate featured image     | OpenAI LangChain               | Generates featured image for article         | Get output                  | Upload featured image to Drive, Resize featured image |                                                                                              |
| Upload featured image to Drive | Google Drive                | Uploads featured image to Drive               | Generate featured image      | Merge2                      |                                                                                              |
| Resize featured image       | Edit Image                    | Resizes featured image                        | Upload featured image to Drive | Merge2                      |                                                                                              |
| Upload featured image       | HTTP Request                  | Uploads featured image to WordPress           | Merge2                      | Update featured image meta data |                                                                                              |
| Update featured image meta data | HTTP Request              | Updates metadata for featured image           | Upload featured image       | Set featured image for post  |                                                                                              |
| Wait featured image         | Merge                         | Synchronizes featured image upload            | Upload featured image, Post on Wordpress | Merge2                      |                                                                                              |
| Merge chapters title and text | Merge                      | Combines chapter titles and texts             | Get title chapter and content, Loop Over Items | Combine into article        |                                                                                              |
| Combine into article        | Code                          | Assembles full article with links in Markdown | Merge chapters title and text | Final article in Markdown    |                                                                                              |
| Final article in Markdown   | Set                           | Prepares Markdown article                      | Combine into article         | Markdown to HTML             |                                                                                              |
| Markdown to HTML            | Markdown                      | Converts Markdown to HTML                       | Final article in Markdown    | FInal article in HTML        |                                                                                              |
| FInal article in HTML       | Set                           | Prepares HTML for WordPress                      | Markdown to HTML             | Post on Wordpress            |                                                                                              |
| Post on Wordpress           | WordPress                     | Publishes post to WordPress                      | FInal article in HTML        | Wait featured image          |                                                                                              |
| Set featured image for post | HTTP Request                  | Sets featured image for WordPress post           | Update featured image meta data | Set excerpt                 |                                                                                              |
| Set excerpt                 | HTTP Request                  | Sets post excerpt                                | Set featured image for post  | Create Doc                  |                                                                                              |
| Edit Fields                | Set                           | Prepares data for Google Docs backup              | Save texts to Doc            |                             |                                                                                              |
| Create Doc                 | Google Docs                   | Creates Google Doc for article backup              | Set excerpt                  | Save texts to Doc            |                                                                                              |
| Save texts to Doc          | Google Docs                   | Saves article content to Google Doc                | Create Doc                  | Edit Fields                 |                                                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Node:**  
   - Add a **Google Sheets Trigger** node. Configure it to monitor a specific Google Sheet and worksheet for new rows.  
   - Set the trigger to activate on new row addition.

2. **Limit to Last Post:**  
   - Add a **Limit** node connected to the trigger. Configure it to pass only the most recent row to avoid duplicate processing.

3. **Normalize Inputs:**  
   - Add a **Set** node to normalize and prepare input parameters (topic, style, audience, chapters, word count, CTA, etc.) from the Google Sheets data.

4. **Validate Inputs:**  
   - Add an **If** node to check that all required inputs are present and valid.  
   - Configure conditions to verify non-empty and correctly formatted fields.  
   - Connect the true output to the next block; false output can stop the workflow or send an alert.

5. **Retrieve Sitemap for Internal Links:**  
   - Add an **HTTP Request** node to fetch the website sitemap XML.  
   - Add an **XML** node to parse the sitemap and extract URLs.  
   - Add a **SplitOut** node to split URLs into individual items.  
   - Add a **Limit** node to restrict to 20 internal links.  
   - Add an **Aggregate** node to reassemble the limited URLs into a list.

6. **Create Backup Folder:**  
   - Add a **Google Drive** node to create a folder named after the blog post title for storing backups.

7. **Initial Research:**  
   - Add a **LangChain Agent** node configured to perform initial research on the main topic using PerplexityAI or equivalent.  
   - Add an **OpenAI Chat Model** node to assist with research text generation.  
   - Add a **Structured Output Parser** node to parse research results into structured JSON.

8. **Article Planning:**  
   - Add a **Chain LLM** node to plan the article structure based on research.  
   - Add an **OpenAI Chat Model** node to support planning.  
   - Add a **Structured Output Parser** node to parse the article plan.  
   - Add an **If** node to check that the plan is not empty.

9. **Split Article into Chapters:**  
   - Add a **SplitOut** node to split the article plan into individual chapters.

10. **Deep Research and Writing Loop:**  
    - Add a **SplitInBatches** node to loop over each chapter sequentially.  
    - Add a **Set** node to extract chapter title and content placeholders.  
    - Add a **LangChain Agent** node to perform deep research per chapter.  
    - Add an **OpenAI Chat Model** node to assist research.  
    - Add a **Structured Output Parser** node to parse chapter research.  
    - Add a **Chain LLM** node to write chapter content based on research.  
    - Add an **OpenAI Chat Model** node to assist writing.  
    - (Optional) Add a **Wait** node to avoid API rate limits (disabled by default).  
    - Add a **Merge** node to combine chapter outputs.

11. **Image Generation and Processing:**  
    - Add an **OpenAI LangChain** node to generate images for each chapter.  
    - Add a **Google Drive** node to upload chapter images to the backup folder.  
    - Add an **Edit Image** node to resize chapter images.  
    - Add an **HTTP Request** node to update WordPress image metadata.  
    - Add an **OpenAI LangChain** node to generate the featured image.  
    - Add a **Google Drive** node to upload the featured image.  
    - Add an **Edit Image** node to resize the featured image.  
    - Add an **HTTP Request** node to upload the featured image to WordPress.  
    - Add an **HTTP Request** node to update featured image metadata.  
    - Add a **Merge** node to synchronize featured image upload with publishing.

12. **Final Assembly and Publishing:**  
    - Add a **Merge** node to combine chapter titles and texts.  
    - Add a **Code** node to assemble the full article in Markdown, inserting internal links.  
    - Add a **Set** node to prepare the Markdown article.  
    - Add a **Markdown** node to convert Markdown to HTML.  
    - Add a **Set** node to prepare HTML content.  
    - Add a **WordPress** node to publish the article (draft or live).  
    - Add an **HTTP Request** node to set the featured image for the post.  
    - Add an **HTTP Request** node to set the post excerpt.

13. **Backup to Google Docs:**  
    - Add a **Google Docs** node to create a new document for the article.  
    - Add a **Google Docs** node to save the article content to the document.

14. **Connect all nodes according to the described flow**, ensuring proper data passing and error handling.

15. **Configure Credentials:**  
    - OpenAI API credentials for text and image generation.  
    - PerplexityAI API credentials for research.  
    - Google Sheets credentials for trigger.  
    - Google Drive and Google Docs credentials for backup.  
    - WordPress credentials (OAuth2 or application password) for publishing.

16. **Test the workflow with sample data**, starting with low-cost AI models and minimal image generation to verify functionality.

---

### 5. General Notes & Resources

| Note Content                                                                                                                             | Context or Link                                                                                   |
|------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| This workflow is a professional-grade content automation machine designed for real-world blogging and SEO needs.                        | Workflow description and purpose                                                                |
| It uses PerplexityAI for real-time deep research, OpenAI for text and image generation, and integrates tightly with WordPress and Google. | Integration details                                                                              |
| The workflow includes advanced looping and rate-limit handling strategies to optimize API usage.                                         | Workflow design notes                                                                           |
| For better image quality, consider using FLUX.1 model instead of OpenAI’s Dall-E for image generation.                                   | Tips for PROs section                                                                           |
| Triggering via Google Sheets is convenient but can be unreliable; consider scheduled triggers or looping for batch processing.           | Tips for PROs section                                                                           |
| Human-in-the-loop review is possible by publishing posts as drafts first, then finalizing after approval.                                | Tips for PROs section                                                                           |
| Minh Duc TV provides expert consulting on n8n and AI-driven automation solutions.                                                        | https://ducnguyen.cc/contact/                                                                   |
| Workflow template includes two JSON files: main workflow and research tool (PerplexityAI).                                               | Setup instructions                                                                             |
| Workflow images illustrate the architecture and flow for visual reference.                                                               | Attached images in original documentation                                                      |

---

This document provides a comprehensive, structured reference for understanding, reproducing, and modifying the **WordPress Auto-Blogging Pro - with DEEP RESEARCH** workflow. It covers all nodes, their roles, configurations, and interconnections, enabling advanced users and AI agents to work effectively with this powerful content automation system.