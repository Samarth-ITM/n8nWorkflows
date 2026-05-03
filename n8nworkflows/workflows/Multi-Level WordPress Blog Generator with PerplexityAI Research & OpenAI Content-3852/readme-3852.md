Multi-Level WordPress Blog Generator with PerplexityAI Research & OpenAI Content

https://n8nworkflows.xyz/workflows/multi-level-wordpress-blog-generator-with-perplexityai-research---openai-content-3852


# Multi-Level WordPress Blog Generator with PerplexityAI Research & OpenAI Content

### 1. Workflow Overview

This workflow, titled **Multi-Level WordPress Blog Generator with PerplexityAI Research & OpenAI Content**, is an advanced content automation system designed to generate deeply researched, long-form WordPress blog posts automatically. It targets users who want to produce authoritative, data-driven articles with multi-level research (topic → sub-topic → sub-sub-topic) powered by PerplexityAI and OpenAI, including AI-generated images and SEO optimizations.

The workflow is structured into three main logical blocks:

- **1.1 Trigger Flow (Scheduler & Dispatcher):**  
  Handles scheduling and input reception from Google Sheets or form submissions. It reads topics marked as "To Do" and dispatches them one by one to the Main Flow for processing, pacing execution with wait times to manage API rate limits.

- **1.2 Main Flow (Content Generation Engine):**  
  The core engine that receives topic parameters, updates status, fetches internal links from the sitemap, performs initial research, plans the blog structure, conditionally executes either deeper multi-level research or standard chapter-level research, generates images, manages tags and categories, assembles the article, converts it to HTML, backs up content, publishes to WordPress, and updates final statuses.

- **1.3 Research Tool Sub-Flow (PerplexityAI Interaction):**  
  A dedicated sub-workflow that interfaces with the PerplexityAI API to perform online research queries at various stages (topic, chapter, subchapter), returning structured research content to the Main Flow.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger Flow (Scheduler & Dispatcher)

- **Overview:**  
  This block initiates the workflow either on a schedule or via form submission. It reads new topics from a Google Sheet, loops through them, and triggers the Main Flow for each topic sequentially, inserting wait times to avoid API rate limits.

- **Nodes Involved:**  
  - Schedule Trigger  
  - On form submission (Form Trigger)  
  - Google Sheets Create Topics  
  - Google Sheets Create Topic - Add New Topic  
  - Loop Over Topics (SplitInBatches)  
  - Execute Workflow - Create A Topic (calls Main Flow)  
  - Wait  

- **Node Details:**

  - **Schedule Trigger**  
    - Type: Trigger node that fires on a configured time interval (default every 10 minutes).  
    - Role: Periodically checks for new topics to process.  
    - Inputs: None  
    - Outputs: Triggers Google Sheets node.  
    - Edge Cases: Misconfigured schedule or disabled node may stop automation.

  - **On form submission**  
    - Type: Webhook trigger for ad-hoc topic submission via n8n form.  
    - Role: Allows manual or external topic submission.  
    - Inputs: HTTP POST form data.  
    - Outputs: Triggers Google Sheets node to add new topic.  
    - Edge Cases: Webhook URL must be accessible; form fields must match expected schema.

  - **Google Sheets Create Topics**  
    - Type: Google Sheets node (read).  
    - Role: Reads rows with `Status` = "To Do" from the `Create Topic` sheet.  
    - Configuration: Reads specific columns (Topic, Audience, Style, Have Deeper Research, etc.).  
    - Inputs: Trigger from Schedule or Form.  
    - Outputs: Passes topic data to Loop Over Topics.  
    - Edge Cases: Google Sheets API errors, incorrect sheet or column names.

  - **Google Sheets Create Topic - Add New Topic**  
    - Type: Google Sheets node (append).  
    - Role: Adds new topics submitted via form to the `Create Topic` sheet.  
    - Retry on Fail: Enabled with 5s wait between tries.  
    - Inputs: Form data.  
    - Outputs: None (end of form submission branch).  
    - Edge Cases: API quota limits, data validation.

  - **Loop Over Topics**  
    - Type: SplitInBatches node.  
    - Role: Processes each "To Do" topic sequentially.  
    - Inputs: List of topics from Google Sheets.  
    - Outputs: Passes single topic data to Execute Workflow node.  
    - Edge Cases: Large batch sizes may cause memory issues; batch size configurable.

  - **Execute Workflow - Create A Topic**  
    - Type: Execute Workflow node.  
    - Role: Calls the Main Flow, passing the current topic parameters.  
    - Inputs: Single topic data.  
    - Outputs: None (wait node follows).  
    - Edge Cases: Workflow not linked or inactive; parameter mismatch.

  - **Wait**  
    - Type: Wait node.  
    - Role: Pauses execution for a configurable duration (default 1 hour) after dispatching each topic to manage API rate limits and pacing.  
    - Inputs: None  
    - Outputs: Loops back to Loop Over Topics for next topic.  
    - Edge Cases: Long wait times may delay processing; short waits risk API throttling.

---

#### 2.2 Main Flow (Content Generation Engine)

- **Overview:**  
  The Main Flow receives topic parameters, updates status, fetches internal links, performs initial research, plans the blog, conditionally executes multi-level or standard research and writing, generates images, manages tags and categories, assembles and converts the article, backs up content, publishes to WordPress, and updates final statuses.

- **Nodes Involved:**  
  - When Executed by Another Workflow (ExecuteWorkflowTrigger)  
  - Google Sheets In Progress Topic  
  - HTTP Request Get Categories  
  - Edit Fields Categories  
  - Aggregate Categories  
  - Check Inputs (If node)  
  - Get Post Sitemap (HTTP Request)  
  - Get XML File (XML parser)  
  - Split Out Links  
  - Limit Internal Links  
  - Aggregate Internal Links  
  - Initial Research (Langchain Agent)  
  - Research Tool (Tool Workflow Sub-Flow)  
  - Structured Output Parser4 (for Initial Research)  
  - Blog Planner (Langchain Chain LLM)  
  - Structured Output Parser1 (for Blog Planner)  
  - Check empty output (If node)  
  - Create Drive Folder (Google Drive)  
  - Split Out Main Chapters  
  - Switch Deep Research (Switch node)  
  - Loop Over Chapter (SplitInBatches)  
  - Loop Over Chapters2 (SplitInBatches)  
  - Chapter Research (Langchain Agent)  
  - Research Tool2 (Tool Workflow Sub-Flow)  
  - Structured Output Parser (for Chapter Research)  
  - Chapter Copywriter (Langchain Chain LLM)  
  - Structured Output Parser5 (for Chapter Copywriter)  
  - Loop Over Sub-Chapter (SplitInBatches)  
  - Subchapter Researcher (Langchain Agent)  
  - Research Tool1 (Tool Workflow Sub-Flow)  
  - Structured Output Parser3 (for Subchapter Copywriter)  
  - Subchapter Copywriter (Langchain Chain LLM)  
  - Get Subchapters Output  
  - Aggregate (merge subchapter outputs)  
  - Get Title Chapter and Content  
  - Merge Chapters  
  - Aggregate All Chapters  
  - Get Output  
  - Loop Over Items (SplitInBatches)  
  - HTTP Request Open AI Image (chapter images)  
  - Split Out Chapter Image  
  - Filter Chapter Image Data  
  - Convert Data To File  
  - Resize Image  
  - Upload Chapter Images  
  - Update Image Meta Data  
  - Upload Chapter Images To Drive  
  - HTTP Request Open AI Image 1 (featured image)  
  - Split Out Feature Image  
  - Filter Feature Image Data  
  - Convert to File  
  - Resize Featured Image  
  - Upload Featured Image  
  - Upload Featured Image To Drive  
  - Update Featured Image Meta Data  
  - Set Featured Image For Post  
  - Set Excerpt  
  - Post On Wordpress  
  - Loop Over Tags (SplitInBatches)  
  - HTTP Request Get Categories1  
  - If Tag Already Exists (If node)  
  - HTTP Request Create Tag  
  - If New Tag (If node)  
  - Edit Fields, Edit Fields2, Edit Fields Final Tags  
  - Aggregate Tag IDs  
  - Merge Tags  
  - Combine Into Article (Code node)  
  - Final Article In Markdown (Set node)  
  - Markdown To HTML  
  - FInal Article In HTML (Set node)  
  - Create Doc (Google Docs)  
  - Save Texts To Doc (Google Docs)  
  - Google Sheets Final Blog  
  - Google Sheets Status Update  

- **Node Details:**

  - **When Executed by Another Workflow**  
    - Type: ExecuteWorkflowTrigger node.  
    - Role: Entry point triggered by the Trigger Flow.  
    - Inputs: Topic parameters from Trigger Flow.  
    - Outputs: Passes data to Google Sheets In Progress Topic.  
    - Edge Cases: Must be linked correctly; inactive node blocks flow.

  - **Google Sheets In Progress Topic**  
    - Type: Google Sheets node (update).  
    - Role: Updates the topic's status to "In Progress" in the `Create Topic` sheet.  
    - Inputs: Topic data.  
    - Outputs: Passes data to HTTP Request Get Categories.  
    - Edge Cases: API errors, incorrect row numbers or sheet names.

  - **HTTP Request Get Categories**  
    - Type: HTTP Request node.  
    - Role: Fetches existing WordPress categories via REST API.  
    - Inputs: None (uses credentials and URL).  
    - Outputs: Category data for processing.  
    - Edge Cases: Auth errors, API downtime.

  - **Edit Fields Categories**  
    - Type: Set node.  
    - Role: Formats and filters category data for later matching.  
    - Inputs: Category data.  
    - Outputs: Passes to Aggregate Categories.  
    - Edge Cases: Data format mismatches.

  - **Aggregate Categories**  
    - Type: Aggregate node.  
    - Role: Aggregates categories into a usable list for matching.  
    - Inputs: Edited category data.  
    - Outputs: Passes to Check Inputs.  
    - Edge Cases: Empty category list.

  - **Check Inputs**  
    - Type: If node.  
    - Role: Validates required inputs like topic, sitemap URL, etc.  
    - Inputs: Aggregated categories and topic data.  
    - Outputs: Branches to Get Post Sitemap or error handling.  
    - Edge Cases: Missing or invalid inputs.

  - **Get Post Sitemap**  
    - Type: HTTP Request node.  
    - Role: Downloads the website's sitemap XML file.  
    - Inputs: Sitemap URL from topic parameters.  
    - Outputs: Passes XML content to Get XML File.  
    - Edge Cases: Sitemap inaccessible or malformed.

  - **Get XML File**  
    - Type: XML node.  
    - Role: Parses the sitemap XML to extract URLs.  
    - Inputs: Sitemap XML content.  
    - Outputs: Passes URLs to Split Out Links.  
    - Edge Cases: Parsing errors.

  - **Split Out Links**  
    - Type: SplitOut node.  
    - Role: Splits URLs into individual items.  
    - Inputs: Parsed URLs.  
    - Outputs: Passes to Limit Internal Links.  
    - Edge Cases: Empty URL list.

  - **Limit Internal Links**  
    - Type: Limit node.  
    - Role: Restricts the number of internal URLs (e.g., max 50) for performance.  
    - Inputs: URL list.  
    - Outputs: Passes to Aggregate Internal Links.  
    - Edge Cases: Limit too low or too high affects linking quality.

  - **Aggregate Internal Links**  
    - Type: Aggregate node.  
    - Role: Aggregates limited URLs into a list for internal linking.  
    - Inputs: Limited URLs.  
    - Outputs: Passes to Check Inputs node for further processing.  
    - Edge Cases: Empty list.

  - **Initial Research**  
    - Type: Langchain Agent node.  
    - Role: Performs initial online research on the main topic using PerplexityAI via the Research Tool Sub-Flow.  
    - Inputs: Topic parameters.  
    - Outputs: Passes research data to Wait6.  
    - Edge Cases: API failures, empty research results.

  - **Research Tool**  
    - Type: Tool Workflow node.  
    - Role: Calls the Research Tool Sub-Flow to interact with PerplexityAI API.  
    - Inputs: Research query parameters.  
    - Outputs: Research content.  
    - Edge Cases: API key errors, rate limits.

  - **Structured Output Parser4**  
    - Type: Langchain Output Parser Structured node.  
    - Role: Parses structured research output from Initial Research.  
    - Inputs: Raw research data.  
    - Outputs: Parsed structured data for Blog Planner.  
    - Edge Cases: Parsing failures.

  - **Blog Planner**  
    - Type: Langchain Chain LLM node.  
    - Role: Uses AI to outline the article: title, subtitle, intro, slug, meta description, tags, chapters, and image prompts.  
    - Inputs: Parsed research data.  
    - Outputs: Passes to Check empty output.  
    - Edge Cases: AI output errors, incomplete outlines.

  - **Structured Output Parser1**  
    - Type: Langchain Output Parser Structured node.  
    - Role: Parses Blog Planner output into structured fields.  
    - Inputs: Raw AI output.  
    - Outputs: Passes to Check empty output.  
    - Edge Cases: Parsing errors.

  - **Check empty output**  
    - Type: If node.  
    - Role: Validates Blog Planner output is not empty before proceeding.  
    - Inputs: Parsed blog plan.  
    - Outputs: Branches to Create Drive Folder and further processing or error handling.  
    - Edge Cases: Empty or invalid output.

  - **Create Drive Folder**  
    - Type: Google Drive node.  
    - Role: Creates a dedicated folder for storing images and backups for the current topic.  
    - Inputs: Topic name or ID.  
    - Outputs: Passes to Get Output.  
    - Edge Cases: Permission errors, folder name conflicts.

  - **Get Output**  
    - Type: Set node.  
    - Role: Prepares data for chapter processing, splitting chapters and tags.  
    - Inputs: Blog Planner output, Drive folder info.  
    - Outputs: Splits chapters and tags for looping.  
    - Edge Cases: Missing chapters or tags.

  - **Split Out Main Chapters**  
    - Type: SplitOut node.  
    - Role: Splits chapters into individual items for looping.  
    - Inputs: Chapters list.  
    - Outputs: Passes to Switch Deep Research.  
    - Edge Cases: Empty chapters.

  - **Switch Deep Research**  
    - Type: Switch node.  
    - Role: Routes flow based on `Have Deeper Research` flag: YES triggers multi-level research, NO triggers standard research.  
    - Inputs: Topic parameter.  
    - Outputs: Branches to Loop Over Chapter (standard) or Loop Over Chapters2 (deeper).  
    - Edge Cases: Invalid flag values.

  - **Loop Over Chapter**  
    - Type: SplitInBatches node.  
    - Role: Iterates over chapters for standard research and writing.  
    - Inputs: Chapters from Switch.  
    - Outputs: Passes to Chapter Research.  
    - Edge Cases: Large chapter lists.

  - **Loop Over Chapters2**  
    - Type: SplitInBatches node.  
    - Role: Iterates over chapters for deeper research path.  
    - Inputs: Chapters from Switch.  
    - Outputs: Passes to Get Title Chapter And Content1 and Main Chapter Researcher.  
    - Edge Cases: Large chapter lists.

  - **Chapter Research**  
    - Type: Langchain Agent node.  
    - Role: Performs research on each chapter using Research Tool Sub-Flow.  
    - Inputs: Chapter data.  
    - Outputs: Passes to Wait3.  
    - Edge Cases: API errors, empty results.

  - **Research Tool2**  
    - Type: Tool Workflow node.  
    - Role: Calls Research Tool Sub-Flow for chapter research.  
    - Inputs: Chapter query.  
    - Outputs: Research content.  
    - Edge Cases: API failures.

  - **Structured Output Parser**  
    - Type: Langchain Output Parser Structured node.  
    - Role: Parses chapter research output.  
    - Inputs: Raw research data.  
    - Outputs: Passes to Chapter Copywriter.  
    - Edge Cases: Parsing errors.

  - **Chapter Copywriter**  
    - Type: Langchain Chain LLM node.  
    - Role: Writes chapter content based on research.  
    - Inputs: Parsed chapter research.  
    - Outputs: Passes to Wait5.  
    - Edge Cases: AI output errors.

  - **Structured Output Parser5**  
    - Type: Langchain Output Parser Structured node.  
    - Role: Parses chapter copywriter output.  
    - Inputs: Raw AI output.  
    - Outputs: Passes to Loop Over Sub-Chapter or aggregation.  
    - Edge Cases: Parsing failures.

  - **Loop Over Sub-Chapter**  
    - Type: SplitInBatches node.  
    - Role: Iterates over subchapters for deeper research path.  
    - Inputs: Subchapter list.  
    - Outputs: Passes to Subchapter Researcher.  
    - Edge Cases: Large subchapter lists.

  - **Subchapter Researcher**  
    - Type: Langchain Agent node.  
    - Role: Performs research on each subchapter.  
    - Inputs: Subchapter data.  
    - Outputs: Passes to Wait4.  
    - Edge Cases: API errors.

  - **Research Tool1**  
    - Type: Tool Workflow node.  
    - Role: Calls Research Tool Sub-Flow for subchapter research.  
    - Inputs: Subchapter query.  
    - Outputs: Research content.  
    - Edge Cases: API failures.

  - **Structured Output Parser3**  
    - Type: Langchain Output Parser Structured node.  
    - Role: Parses subchapter copywriter output.  
    - Inputs: Raw AI output.  
    - Outputs: Passes to Subchapter Copywriter.  
    - Edge Cases: Parsing errors.

  - **Subchapter Copywriter**  
    - Type: Langchain Chain LLM node.  
    - Role: Writes detailed content for each subchapter.  
    - Inputs: Parsed research.  
    - Outputs: Passes to Wait2.  
    - Edge Cases: AI output errors.

  - **Get Subchapters Output**  
    - Type: Set node.  
    - Role: Prepares subchapter content for aggregation.  
    - Inputs: Subchapter copywriter output.  
    - Outputs: Passes to Split Out Subchapters.  
    - Edge Cases: Empty content.

  - **Aggregate**  
    - Type: Aggregate node.  
    - Role: Combines subchapter contents into full chapter content.  
    - Inputs: Subchapter outputs.  
    - Outputs: Passes to Get Title Chapter and Content.  
    - Edge Cases: Missing subchapter data.

  - **Get Title Chapter and Content**  
    - Type: Set node.  
    - Role: Extracts chapter title and aggregated content for final assembly.  
    - Inputs: Aggregated subchapter content.  
    - Outputs: Passes to Merge Chapters.  
    - Edge Cases: Missing titles or content.

  - **Merge Chapters**  
    - Type: Merge node.  
    - Role: Merges chapter contents into a complete article structure.  
    - Inputs: Chapter contents.  
    - Outputs: Passes to Aggregate All Chapters.  
    - Edge Cases: Merge conflicts.

  - **Aggregate All Chapters**  
    - Type: Aggregate node.  
    - Role: Aggregates all chapters into a single article data object.  
    - Inputs: Merged chapters.  
    - Outputs: Passes to Blog Planner for final assembly.  
    - Edge Cases: Empty chapters.

  - **Get Output**  
    - Type: Set node.  
    - Role: Prepares data for image generation and tag processing.  
    - Inputs: Aggregated article data.  
    - Outputs: Splits tags and chapters for further processing.  
    - Edge Cases: Missing tags or chapters.

  - **Loop Over Items**  
    - Type: SplitInBatches node.  
    - Role: Loops over chapters for image generation and article assembly.  
    - Inputs: Chapters.  
    - Outputs: Passes to Combine Into Article and HTTP Request Open AI Image.  
    - Edge Cases: Large chapter lists.

  - **HTTP Request Open AI Image**  
    - Type: HTTP Request node.  
    - Role: Calls OpenAI API to generate images for chapters using `gpt-image-1` or equivalent.  
    - Inputs: Image prompts from chapters.  
    - Outputs: Passes to Split Out Chapter Image.  
    - Edge Cases: API rate limits, image generation failures.

  - **Split Out Chapter Image**  
    - Type: SplitOut node.  
    - Role: Splits image data for individual processing.  
    - Inputs: Image generation response.  
    - Outputs: Passes to Filter Chapter Image Data.  
    - Edge Cases: Empty images.

  - **Filter Chapter Image Data**  
    - Type: Filter node.  
    - Role: Filters valid image data for upload.  
    - Inputs: Split images.  
    - Outputs: Passes to Convert Data To File.  
    - Edge Cases: Invalid image data.

  - **Convert Data To File**  
    - Type: ConvertToFile node.  
    - Role: Converts image data to file format for upload.  
    - Inputs: Filtered image data.  
    - Outputs: Passes to Resize Image.  
    - Edge Cases: Conversion errors.

  - **Resize Image**  
    - Type: EditImage node.  
    - Role: Resizes images to configured dimensions.  
    - Inputs: Image files.  
    - Outputs: Passes to Upload Chapter Images.  
    - Edge Cases: Image processing errors.

  - **Upload Chapter Images**  
    - Type: HTTP Request node.  
    - Role: Uploads images to WordPress Media Library.  
    - Inputs: Resized images.  
    - Outputs: Passes to Update Image Meta Data.  
    - Edge Cases: Upload failures, auth errors.

  - **Update Image Meta Data**  
    - Type: HTTP Request node.  
    - Role: Updates metadata for uploaded images in WordPress.  
    - Inputs: Upload response.  
    - Outputs: Passes to Upload Chapter Images To Drive.  
    - Edge Cases: API errors.

  - **Upload Chapter Images To Drive**  
    - Type: Google Drive node.  
    - Role: Saves images to Google Drive backup folder.  
    - Inputs: Image files.  
    - Outputs: Passes to Merge1.  
    - Edge Cases: Drive permission errors.

  - **HTTP Request Open AI Image 1**  
    - Type: HTTP Request node.  
    - Role: Generates featured image for the article.  
    - Inputs: Featured image prompt.  
    - Outputs: Passes to Split Out Feature Image.  
    - Edge Cases: API errors.

  - **Split Out Feature Image**  
    - Type: SplitOut node.  
    - Role: Splits featured image data.  
    - Inputs: Image generation response.  
    - Outputs: Passes to Filter Feature Image Data.  
    - Edge Cases: Empty data.

  - **Filter Feature Image Data**  
    - Type: Filter node.  
    - Role: Filters valid featured image data.  
    - Inputs: Split images.  
    - Outputs: Passes to Convert to File.  
    - Edge Cases: Invalid data.

  - **Convert to File**  
    - Type: ConvertToFile node.  
    - Role: Converts featured image data to file.  
    - Inputs: Filtered data.  
    - Outputs: Passes to Resize Featured Image.  
    - Edge Cases: Conversion errors.

  - **Resize Featured Image**  
    - Type: EditImage node.  
    - Role: Resizes featured image.  
    - Inputs: File.  
    - Outputs: Passes to Upload Featured Image.  
    - Edge Cases: Processing errors.

  - **Upload Featured Image**  
    - Type: HTTP Request node.  
    - Role: Uploads featured image to WordPress Media Library.  
    - Inputs: Resized image.  
    - Outputs: Passes to Update Featured Image Meta Data.  
    - Edge Cases: Upload failures.

  - **Upload Featured Image To Drive**  
    - Type: Google Drive node.  
    - Role: Saves featured image to Google Drive.  
    - Inputs: Image file.  
    - Outputs: Passes to Merge2.  
    - Edge Cases: Permission errors.

  - **Update Featured Image Meta Data**  
    - Type: HTTP Request node.  
    - Role: Updates metadata for featured image in WordPress.  
    - Inputs: Upload response.  
    - Outputs: Passes to Set Featured Image For Post.  
    - Edge Cases: API errors.

  - **Set Featured Image For Post**  
    - Type: HTTP Request node.  
    - Role: Assigns the featured image to the WordPress post.  
    - Inputs: Post ID and image ID.  
    - Outputs: Passes to Set Excerpt.  
    - Edge Cases: API errors.

  - **Set Excerpt**  
    - Type: HTTP Request node.  
    - Role: Sets the post excerpt and meta description.  
    - Inputs: Post ID and excerpt text.  
    - Outputs: Passes to Google Sheets Final Blog.  
    - Edge Cases: API errors.

  - **Post On Wordpress**  
    - Type: WordPress node.  
    - Role: Creates the WordPress post with title, content (HTML), slug, category, tags, and featured image.  
    - Inputs: Final article data, tags, categories, images.  
    - Outputs: Passes to Wait Featured Image.  
    - Edge Cases: Auth errors, API limits.

  - **Loop Over Tags**  
    - Type: SplitInBatches node.  
    - Role: Iterates over AI-generated tags for validation and creation.  
    - Inputs: Tag list.  
    - Outputs: Passes to If Tag Already Exists.  
    - Edge Cases: Large tag lists.

  - **HTTP Request Get Categories1**  
    - Type: HTTP Request node.  
    - Role: Retrieves existing WordPress tags.  
    - Inputs: None.  
    - Outputs: Passes to If Tag Already Exists.  
    - Edge Cases: API errors.

  - **If Tag Already Exists**  
    - Type: If node.  
    - Role: Checks if a tag exists in WordPress.  
    - Inputs: Tag data.  
    - Outputs: Branches to Edit Fields (existing) or HTTP Request Create Tag (new).  
    - Edge Cases: False negatives/positives.

  - **HTTP Request Create Tag**  
    - Type: HTTP Request node.  
    - Role: Creates new tags in WordPress if they do not exist.  
    - Inputs: Tag name.  
    - Outputs: Passes to If New Tag.  
    - Edge Cases: API errors.

  - **If New Tag**  
    - Type: If node.  
    - Role: Confirms tag creation success.  
    - Inputs: API response.  
    - Outputs: Passes to Edit Fields2.  
    - Edge Cases: Creation failures.

  - **Edit Fields, Edit Fields2, Edit Fields Final Tags**  
    - Type: Set nodes.  
    - Role: Formats tag data and collects tag IDs for post assignment.  
    - Inputs: Tag data.  
    - Outputs: Passes to Aggregate Tag IDs and Merge Tags.  
    - Edge Cases: Missing IDs.

  - **Aggregate Tag IDs**  
    - Type: Aggregate node.  
    - Role: Aggregates all tag IDs into a list for the post.  
    - Inputs: Tag ID data.  
    - Outputs: Passes to Edit Fields Final Tags.  
    - Edge Cases: Empty tag list.

  - **Merge Tags**  
    - Type: Merge node.  
    - Role: Combines tag data with post data before publishing.  
    - Inputs: Tag and post data.  
    - Outputs: Passes to Post On Wordpress.  
    - Edge Cases: Merge conflicts.

  - **Combine Into Article**  
    - Type: Code node.  
    - Role: Assembles the full article in Markdown format, including introduction, chapters, conclusion, CTA, and Table of Contents if applicable.  
    - Inputs: Chapter contents, metadata.  
    - Outputs: Passes to Final Article In Markdown.  
    - Edge Cases: Code errors, malformed Markdown.

  - **Final Article In Markdown**  
    - Type: Set node.  
    - Role: Stores the final Markdown article.  
    - Inputs: Code node output.  
    - Outputs: Passes to Markdown To HTML.  
    - Edge Cases: Empty content.

  - **Markdown To HTML**  
    - Type: Markdown node.  
    - Role: Converts Markdown article to HTML for WordPress.  
    - Inputs: Markdown text.  
    - Outputs: Passes to FInal Article In HTML.  
    - Edge Cases: Conversion errors.

  - **FInal Article In HTML**  
    - Type: Set node.  
    - Role: Stores final HTML content.  
    - Inputs: Converted HTML.  
    - Outputs: Passes to Create Doc.  
    - Edge Cases: Empty content.

  - **Create Doc**  
    - Type: Google Docs node.  
    - Role: Creates a Google Docs document with the Markdown article for backup.  
    - Inputs: Markdown content, Drive folder.  
    - Outputs: Passes to Save Texts To Doc.  
    - Edge Cases: Google Docs API restrictions (only works on My Drive, not Shared Drives).

  - **Save Texts To Doc**  
    - Type: Google Docs node.  
    - Role: Saves or updates the document.  
    - Inputs: Document data.  
    - Outputs: Passes to Merge Tags.  
    - Edge Cases: API errors.

  - **Google Sheets Final Blog**  
    - Type: Google Sheets node (append/update).  
    - Role: Saves final blog details (URL, Markdown, HTML, tags, meta) to the `Final Blogs` sheet.  
    - Inputs: Post data.  
    - Outputs: Passes to Google Sheets Status Update.  
    - Edge Cases: API errors.

  - **Google Sheets Status Update**  
    - Type: Google Sheets node (update).  
    - Role: Updates the original topic's status to "Done" in the `Create Topic` sheet.  
    - Inputs: Topic row number.  
    - Outputs: Ends flow.  
    - Edge Cases: API errors.

---

#### 2.3 Research Tool Sub-Flow (PerplexityAI Interaction)

- **Overview:**  
  This sub-workflow handles all interactions with the PerplexityAI API, performing research queries at different levels (topic, chapter, subchapter) and returning structured research content to the Main Flow.

- **Nodes Involved:**  
  - PerplexityAI API (HTTP Request)  
  - Edit Fields1 (Set)  
  - Get Research Content (Set)  

- **Node Details:**

  - **PerplexityAI API**  
    - Type: HTTP Request node.  
    - Role: Sends POST requests to PerplexityAI API with research queries, model selection (`sonar` or `sonar-deep-research`), system message, and max tokens.  
    - Inputs: Research parameters from Main Flow.  
    - Outputs: Raw API response.  
    - Edge Cases: API key errors, rate limits, network issues.

  - **Edit Fields1**  
    - Type: Set node.  
    - Role: Prepares request payload or parameters for the API call.  
    - Inputs: Incoming research query parameters.  
    - Outputs: Passes to PerplexityAI API node.  
    - Edge Cases: Incorrect parameter formatting.

  - **Get Research Content**  
    - Type: Set node.  
    - Role: Extracts and formats the research content from the API response for return to Main Flow.  
    - Inputs: API response.  
    - Outputs: Returns research content.  
    - Edge Cases: Missing or malformed response data.

---

### 3. Summary Table

| Node Name                         | Node Type                          | Functional Role                         | Input Node(s)                      | Output Node(s)                       | Sticky Note                          |
|----------------------------------|----------------------------------|---------------------------------------|----------------------------------|------------------------------------|------------------------------------|
| Schedule Trigger                 | Schedule Trigger                 | Periodic trigger to start workflow    | None                             | Google Sheets Create Topics         |                                    |
| On form submission              | Form Trigger                    | Webhook trigger for manual topic input| None                             | Google Sheets Create Topic - Add New Topic |                                    |
| Google Sheets Create Topics     | Google Sheets                   | Reads topics with Status = "To Do"    | Schedule Trigger                 | Loop Over Topics                    |                                    |
| Google Sheets Create Topic - Add New Topic | Google Sheets                   | Adds new topics from form submissions | On form submission              | None                               |                                    |
| Loop Over Topics                | SplitInBatches                  | Processes each topic sequentially     | Google Sheets Create Topics      | Execute Workflow - Create A Topic  |                                    |
| Execute Workflow - Create A Topic | Execute Workflow               | Calls Main Flow for topic processing  | Loop Over Topics                 | Wait                              |                                    |
| Wait                           | Wait                           | Pauses between topic processing       | Execute Workflow - Create A Topic| Loop Over Topics                   |                                    |
| When Executed by Another Workflow | ExecuteWorkflowTrigger         | Entry point for Main Flow              | Execute Workflow - Create A Topic| Google Sheets In Progress Topic    |                                    |
| Google Sheets In Progress Topic | Google Sheets                   | Updates topic status to "In Progress" | When Executed by Another Workflow| HTTP Request Get Categories        |                                    |
| HTTP Request Get Categories     | HTTP Request                   | Fetches WordPress categories           | Google Sheets In Progress Topic  | Edit Fields Categories             |                                    |
| Edit Fields Categories          | Set                            | Formats category data                   | HTTP Request Get Categories      | Aggregate Categories               |                                    |
| Aggregate Categories            | Aggregate                      | Aggregates categories                   | Edit Fields Categories           | Check Inputs                      |                                    |
| Check Inputs                   | If                             | Validates inputs                        | Aggregate Categories             | Get Post Sitemap / Error Handling  |                                    |
| Get Post Sitemap               | HTTP Request                   | Downloads sitemap XML                   | Check Inputs                    | Get XML File                     |                                    |
| Get XML File                  | XML                            | Parses sitemap XML                      | Get Post Sitemap                | Split Out Links                  |                                    |
| Split Out Links              | SplitOut                       | Splits URLs                            | Get XML File                   | Limit Internal Links             |                                    |
| Limit Internal Links         | Limit                         | Limits number of internal URLs         | Split Out Links                | Aggregate Internal Links         |                                    |
| Aggregate Internal Links     | Aggregate                     | Aggregates internal URLs                | Limit Internal Links           | Initial Research / Check Inputs  |                                    |
| Initial Research             | Langchain Agent               | Performs initial topic research        | Aggregate Internal Links        | Wait6                          |                                    |
| Research Tool                | Tool Workflow                 | Calls Research Tool Sub-Flow           | Initial Research               | Structured Output Parser4        |                                    |
| Structured Output Parser4    | Output Parser Structured      | Parses initial research output         | Research Tool                 | Blog Planner                   |                                    |
| Blog Planner                | Langchain Chain LLM           | Creates blog outline and metadata      | Structured Output Parser4       | Check empty output             |                                    |
| Structured Output Parser1    | Output Parser Structured      | Parses blog planner output              | Blog Planner                  | Check empty output             |                                    |
| Check empty output           | If                           | Validates blog plan output              | Structured Output Parser1       | Create Drive Folder / Error Handling |                                    |
| Create Drive Folder          | Google Drive                 | Creates folder for backups and images  | Check empty output             | Get Output                    |                                    |
| Get Output                  | Set                          | Prepares chapters and tags for looping | Create Drive Folder            | Split Out Main Chapters / Split Out Tags |                                    |
| Split Out Main Chapters     | SplitOut                     | Splits chapters for processing         | Get Output                   | Switch Deep Research          |                                    |
| Switch Deep Research        | Switch                       | Routes flow based on deeper research flag | Split Out Main Chapters       | Loop Over Chapter / Loop Over Chapters2 |                                    |
| Loop Over Chapter           | SplitInBatches               | Loops chapters for standard research   | Switch Deep Research          | Chapter Research            |                                    |
| Loop Over Chapters2         | SplitInBatches               | Loops chapters for deeper research     | Switch Deep Research          | Get Title Chapter And Content1 / Main Chapter Researcher |                                    |
| Chapter Research            | Langchain Agent             | Performs chapter-level research         | Loop Over Chapter             | Wait3                       |                                    |
| Research Tool2              | Tool Workflow               | Calls Research Tool Sub-Flow for chapters | Chapter Research             | Structured Output Parser       |                                    |
| Structured Output Parser    | Output Parser Structured    | Parses chapter research output          | Research Tool2               | Chapter Copywriter           |                                    |
| Chapter Copywriter          | Langchain Chain LLM         | Writes chapter content                   | Structured Output Parser      | Wait5                       |                                    |
| Structured Output Parser5   | Output Parser Structured    | Parses chapter copywriter output         | Chapter Copywriter           | Loop Over Sub-Chapter / Aggregation |                                    |
| Loop Over Sub-Chapter       | SplitInBatches             | Loops subchapters for deeper research   | Structured Output Parser5    | Subchapter Researcher       |                                    |
| Subchapter Researcher       | Langchain Agent           | Performs subchapter-level research       | Loop Over Sub-Chapter        | Wait4                       |                                    |
| Research Tool1              | Tool Workflow             | Calls Research Tool Sub-Flow for subchapters | Subchapter Researcher        | Structured Output Parser3      |                                    |
| Structured Output Parser3   | Output Parser Structured  | Parses subchapter copywriter output      | Research Tool1               | Subchapter Copywriter       |                                    |
| Subchapter Copywriter       | Langchain Chain LLM       | Writes subchapter content                 | Structured Output Parser3    | Wait2                       |                                    |
| Get Subchapters Output      | Set                      | Prepares subchapter content for aggregation | Subchapter Copywriter       | Split Out Subchapters        |                                    |
| Aggregate                  | Aggregate                 | Aggregates subchapter contents            | Get Subchapters Output       | Get Title Chapter and Content |                                    |
| Get Title Chapter and Content | Set                      | Extracts chapter title and content        | Aggregate                   | Merge Chapters              |                                    |
| Merge Chapters             | Merge                    | Merges chapter contents                   | Get Title Chapter and Content | Aggregate All Chapters       |                                    |
| Aggregate All Chapters     | Aggregate                 | Aggregates all chapters into article       | Merge Chapters              | Blog Planner                |                                    |
| Loop Over Items            | SplitInBatches           | Loops over chapters for image generation and assembly | Split Out Chapters          | Combine Into Article / HTTP Request Open AI Image |                                    |
| HTTP Request Open AI Image | HTTP Request             | Generates chapter images via OpenAI API   | Loop Over Items             | Split Out Chapter Image      |                                    |
| Split Out Chapter Image    | SplitOut                 | Splits chapter image data                  | HTTP Request Open AI Image  | Filter Chapter Image Data    |                                    |
| Filter Chapter Image Data  | Filter                   | Filters valid chapter images               | Split Out Chapter Image     | Convert Data To File         |                                    |
| Convert Data To File       | ConvertToFile            | Converts image data to file                 | Filter Chapter Image Data   | Resize Image                |                                    |
| Resize Image              | EditImage                | Resizes chapter images                      | Convert Data To File        | Upload Chapter Images       |                                    |
| Upload Chapter Images     | HTTP Request             | Uploads chapter images to WordPress         | Resize Image               | Update Image Meta Data      |                                    |
| Update Image Meta Data    | HTTP Request             | Updates metadata for uploaded images        | Upload Chapter Images      | Upload Chapter Images To Drive |                                    |
| Upload Chapter Images To Drive | Google Drive             | Saves chapter images to Google Drive        | Update Image Meta Data     | Merge1                     |                                    |
| HTTP Request Open AI Image 1 | HTTP Request             | Generates featured image                      | Get Output                 | Split Out Feature Image     |                                    |
| Split Out Feature Image   | SplitOut                 | Splits featured image data                   | HTTP Request Open AI Image 1 | Filter Feature Image Data   |                                    |
| Filter Feature Image Data | Filter                   | Filters valid featured images                 | Split Out Feature Image    | Convert to File             |                                    |
| Convert to File           | ConvertToFile            | Converts featured image data to file          | Filter Feature Image Data  | Resize Featured Image       |                                    |
| Resize Featured Image     | EditImage                | Resizes featured image                         | Convert to File            | Upload Featured Image       |                                    |
| Upload Featured Image     | HTTP Request             | Uploads featured image to WordPress            | Resize Featured Image      | Update Featured Image Meta Data |                                    |
| Upload Featured Image To Drive | Google Drive             | Saves featured image to Google Drive           | Upload Featured Image      | Merge2                     |                                    |
| Update Featured Image Meta Data | HTTP Request             | Updates metadata for featured image            | Upload Featured Image      | Set Featured Image For Post |                                    |
| Set Featured Image For Post | HTTP Request             | Assigns featured image to WordPress post         | Update Featured Image Meta Data | Set Excerpt                |                                    |
| Set Excerpt               | HTTP Request             | Sets post excerpt and meta description           | Set Featured Image For Post | Google Sheets Final Blog    |                                    |
| Post On Wordpress         | WordPress                | Creates WordPress post with content and metadata | Merge Tags                 | Wait Featured Image         |                                    |
| Loop Over Tags            | SplitInBatches           | Loops over tags for validation and creation       | Split Out Tags             | If Tag Already Exists       |                                    |
| HTTP Request Get Categories1 | HTTP Request             | Fetches existing WordPress tags                   | Loop Over Tags             | If Tag Already Exists       |                                    |
| If Tag Already Exists     | If                       | Checks if tag exists in WordPress                   | HTTP Request Get Categories1 | Edit Fields / HTTP Request Create Tag |                                    |
| HTTP Request Create Tag   | HTTP Request             | Creates new WordPress tags                         | If Tag Already Exists      | If New Tag                  |                                    |
| If New Tag                | If                       | Confirms new tag creation                          | HTTP Request Create Tag    | Edit Fields2                |                                    |
| Edit Fields              | Set                      | Formats existing tag data                           | If Tag Already Exists      | Loop Over Tags              |                                    |
| Edit Fields2             | Set                      | Formats new tag data                                | If New Tag                 | Loop Over Tags              |                                    |
| Edit Fields Final Tags   | Set                      | Prepares final tag ID list                           | Aggregate Tag IDs          | Merge Tags                  |                                    |
| Aggregate Tag IDs        | Aggregate                 | Aggregates all tag IDs                              | Loop Over Tags             | Edit Fields Final Tags      |                                    |
| Merge Tags               | Merge                    | Merges tag data with post data                      | Edit Fields Final Tags     | Post On Wordpress           |                                    |
| Combine Into Article     | Code                     | Assembles full article in Markdown                   | Loop Over Items            | Final Article In Markdown   |                                    |
| Final Article In Markdown | Set                      | Stores final Markdown article                        | Combine Into Article       | Markdown To HTML            |                                    |
| Markdown To HTML         | Markdown                 | Converts Markdown to HTML                            | Final Article In Markdown  | FInal Article In HTML       |                                    |
| FInal Article In HTML    | Set                      | Stores final HTML content                            | Markdown To HTML           | Create Doc                  |                                    |
| Create Doc               | Google Docs              | Creates Google Docs document for backup             | FInal Article In HTML      | Save Texts To Doc           |                                    |
| Save Texts To Doc        | Google Docs              | Saves or updates Google Docs document                | Create Doc                 | Merge Tags                  |                                    |
| Google Sheets Final Blog | Google Sheets            | Saves final blog details to `Final Blogs` sheet      | Set Excerpt                | Google Sheets Status Update |                                    |
| Google Sheets Status Update | Google Sheets            | Updates original topic status to "Done"              | Google Sheets Final Blog   | None                       |                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Flow:**

   - Add a **Schedule Trigger** node with a desired interval (e.g., every 10 minutes).
   - Add a **Form Trigger** node for manual topic submission.
   - Add a **Google Sheets** node configured to read rows from the `Create Topic` sheet where `Status` = "To Do".
   - Add a **Google Sheets** node to append new topics from the form submission.
   - Add a **SplitInBatches** node to loop over topics read from Google Sheets.
   - Add an **Execute Workflow** node configured to call the Main Flow, passing topic data.
   - Add a **Wait** node (default 1 hour) after the Execute Workflow node to pace processing.
   - Connect nodes: Schedule Trigger → Google Sheets Create Topics → Loop Over Topics → Execute Workflow → Wait → Loop Over Topics; Form Trigger → Google Sheets Create Topic - Add New Topic.

2. **Create Main Flow:**

   - Add an **ExecuteWorkflowTrigger** node as entry point.
   - Add a **Google Sheets** node to update topic status to "In Progress" in `Create Topic`.
   - Add an **HTTP Request** node to fetch WordPress categories.
   - Add **Set** and **Aggregate** nodes to format and aggregate categories.
   - Add an **If** node to validate inputs (topic, sitemap URL, etc.).
   - Add an **HTTP Request** node to download sitemap XML.
   - Add an **XML** node to parse sitemap.
   - Add **SplitOut**, **Limit**, and **Aggregate** nodes to extract and limit internal links.
   - Add a **Langchain Agent** node (Initial Research) calling the Research Tool Sub-Flow.
   - Add a **Tool Workflow** node configured to call the Research Tool Sub-Flow.
   - Add **Output Parser Structured** nodes to parse AI outputs.
   - Add a **Langchain Chain LLM** node (Blog Planner) to generate article outline.
   - Add an **If** node to check Blog Planner output.
   - Add a **Google Drive** node to create a folder for backups.
   - Add **Set** and **SplitOut** nodes to prepare chapters and tags.
   - Add a **Switch** node to route based on `Have Deeper Research` flag.
   - For standard research path: add **SplitInBatches** (Loop Over Chapter), **Langchain Agent** (Chapter Research), **Tool Workflow** (Research Tool Sub-Flow), **Output Parser**, **Langchain Chain LLM** (Chapter Copywriter), and related **Wait** nodes.
   - For deeper research path: add **SplitInBatches** (Loop Over Chapters2), **Langchain Agent** (Main Chapter Researcher), **SplitInBatches** (Loop Over Sub-Chapter), **Langchain Agent** (Subchapter Researcher), **Tool Workflow** (Research Tool Sub-Flow), **Output Parser**, **Langchain Chain LLM** (Subchapter Copywriter), and related **Wait** nodes.
   - Add **Aggregate** and **Merge** nodes to combine subchapter and chapter content.
   - Add **SplitInBatches** (Loop Over Items) for image generation and article assembly.
   - Add **HTTP Request** nodes to call OpenAI image generation API for chapters and featured image.
   - Add **SplitOut**, **Filter**, **ConvertToFile**, **EditImage**, **HTTP Request**, and **Google Drive** nodes to process and upload images.
   - Add **WordPress** node to create the post with content, tags, categories, and featured image.
   - Add **SplitInBatches** and **HTTP Request** nodes to manage tags: check existing tags, create new tags, and aggregate tag IDs.
   - Add **Code** node to assemble final article in Markdown.
   - Add **Markdown** node to convert Markdown to HTML.
   - Add **Google Docs** nodes to save Markdown backup.
   - Add **Google Sheets** nodes to save final blog details and update topic status to "Done".
   - Connect all nodes respecting the logical flow described in section 2.2.

3. **Create Research Tool Sub-Flow:**

   - Add a **Set** node to prepare API request parameters.
   - Add an **HTTP Request** node configured to call PerplexityAI API with authentication.
   - Add a **Set** node to extract and format research content from API response.
   - Configure inputs and outputs to receive query parameters and return research content.
   - Link this sub-workflow to the Main Flow's Tool Workflow nodes.

4. **Configure Credentials:**

   - Add OpenAI credentials for text and image generation.
   - Add PerplexityAI API credentials.
   - Add WordPress credentials with Application Password for REST API access.
   - Add Google credentials for Sheets, Drive, and Docs access.

5. **Set Parameters and Defaults:**

   - In Google Sheets nodes, specify correct workbook and sheet names (`Create Topic`, `Final Blogs`).
   - In HTTP Request nodes, set correct URLs (WordPress REST endpoints, sitemap URL).
   - In Langchain nodes, configure prompts and models (default GPT-4o for text, `gpt-image-1` for images).
   - Set batch sizes and wait durations to manage API limits.
   - Configure Switch node to read `Have Deeper Research` flag from topic data.

6. **Test and Activate:**

   - Test each node individually to verify credentials and connectivity.
   - Run the Trigger Flow manually with a test topic.
   - Verify WordPress post creation, Google Docs backup, and Google Sheets status updates.
   - Activate the Trigger Flow; keep Main Flow and Research Tool Sub-Flow inactive (triggered by other flows).

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                 | Context or Link                                                                                  |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| This workflow is an upgraded version of v1: [WordPress Auto-Blogging Pro - with DEEP RESEARCH v1](https://n8n.io/workflows/3041-wordpress-auto-blogging-pro-with-deep-research-content-automation-machine/)                      | Upgrade reference                                                                              |
| Visit AI Automation Pro’s website for more powerful n8n templates: https://aiautomationpro.org/                                                                                                                             | Official template provider                                                                     |
| Google Docs backup only works on "My Drive" and will NOT work on Shared Drives due to Google Drive API restrictions.                                                                                                         | Important Google Docs API limitation                                                           |
| Use the `Have Deeper Research` flag in Google Sheets to control research depth and optimize API usage.                                                                                                                       | Cost and complexity management                                                                 |
| Ensure your WordPress `post-sitemap.xml` is publicly accessible and correctly formatted for internal link extraction.                                                                                                        | Sitemap accessibility requirement                                                              |
| Adjust wait times in Trigger Flow and Main Flow to avoid API rate limits for OpenAI and PerplexityAI.                                                                                                                        | API rate limit management                                                                      |
| For image generation, alternative models like Flux.1 can be used by modifying HTTP Request nodes.                                                                                                                           | Image generation flexibility                                                                   |
| For deeper research, test PerplexityAI's `sonar-deep-research` model by passing it as a parameter in the Research Tool Sub-Flow.                                                                                            | Research model customization                                                                   |
| Contact AI Automation Pro for support or inquiries: https://aiautomationpro.org/contact/                                                                                                                                     | Support contact                                                                               |

---

This documentation provides a complete, structured reference for understanding, reproducing, and modifying the Multi-Level WordPress Blog Generator workflow with PerplexityAI and OpenAI integration. It anticipates potential failure points and integration considerations for robust operation.