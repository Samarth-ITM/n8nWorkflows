WordPress Blog Automation with Airtable Interface, Human Review & AI Research v2

https://n8nworkflows.xyz/workflows/wordpress-blog-automation-with-airtable-interface--human-review---ai-research-v2-7961


# WordPress Blog Automation with Airtable Interface, Human Review & AI Research v2

### 1. Workflow Overview

This workflow automates content creation, management, and publication for a WordPress blog using Airtable as the interface and storage system. It integrates human review steps and deep AI-driven research and copywriting to generate high-quality, structured blog content. The workflow handles topic creation, chapter and subchapter generation, featured image management, and final post publication on WordPress.

The workflow is organized into the following logical blocks:

- **1.1 Topic Management & Input Reception:** Detects new topics via Airtable triggers, checks their status and confirmation, and prepares for processing.
- **1.2 Content Planning & Research:** Runs AI-powered blog planning and initial research, including deep research options and external data retrieval (e.g., sitemap and categories).
- **1.3 Content Generation & Chapter Handling:** Generates chapters and subchapters via AI agents, manages batch processing, and handles content updates in Airtable.
- **1.4 Image Processing & Upload:** Automates image generation, resizing, uploading to Google Drive, and metadata updates, including manual image handling.
- **1.5 Content Finalization & Publication:** Combines generated content and images, converts Markdown to HTML, updates Airtable with final content, and publishes posts on WordPress.
- **1.6 Cleanup & Status Updates:** Deletes processed records after completion, updates statuses in Airtable, and manages wait states to synchronize processing.

---

### 2. Block-by-Block Analysis

#### 2.1 Topic Management & Input Reception

**Overview:**  
This block listens for new or updated topics in Airtable, verifies their status and confirmation, and retrieves relevant topic data to initiate content processing.

**Nodes Involved:**  
- Create New Topic (Airtable Trigger)  
- Airtable Get Topic  
- Split Out All Records  
- Filter Status  
- Limit To First Item  
- Check Status & Confirmation (If)  
- Confirm Topic (Airtable)  
- Wait To Update New Topic  
- Settings (Code)  

**Node Details:**

- **Create New Topic**  
  - Type: Airtable Trigger  
  - Role: Listens for new topic entries in Airtable to trigger workflow start.  
  - Input: Airtable topic changes  
  - Output: New topic data  
  - Notes: Critical for starting topic processing.

- **Airtable Get Topic**  
  - Type: Airtable Read  
  - Role: Retrieves detailed topic records from Airtable for further processing.  
  - Input: Trigger from "Create New Topic"  
  - Output: Topic records array  
  - Potential Failures: Airtable API limits, authentication errors.

- **Split Out All Records**  
  - Type: SplitOut  
  - Role: Splits array of topic records into individual items for filtering.  
  - Input: Topics array  
  - Output: Single topic records  

- **Filter Status**  
  - Type: Filter  
  - Role: Filters topics by status to process only appropriate ones (e.g., "In Progress").  
  - Input: Individual topic record  
  - Output: Filtered topic records  

- **Limit To First Item**  
  - Type: Limit  
  - Role: Limits processing to the first matching topic to avoid parallel conflicts.  
  - Input: Filtered topic records  
  - Output: Single topic record  

- **Check Status & Confirmation (If)**  
  - Type: If  
  - Role: Checks if a topic is confirmed and ready for processing.  
  - Input: Single topic record  
  - Output: Branches “True” or “False” to control flow  
  - Edge Cases: Missing confirmation flag, expression errors.

- **Confirm Topic**  
  - Type: Airtable Update  
  - Role: Marks topic as confirmed or updates status in Airtable.  
  - Input: True branch from Check Status & Confirmation  
  - Output: Updated Airtable record  

- **Wait To Update New Topic**  
  - Type: Wait (Webhook)  
  - Role: Waits for external confirmation or delay before continuing processing.  
  - Input: After Confirm Topic  
  - Output: Continues flow post-wait  

- **Settings**  
  - Type: Code  
  - Role: Stores or computes configuration variables or constants for downstream use.  
  - Input/Output: Internal to workflow  
  - Notes: Central place for runtime configurations.

---

#### 2.2 Content Planning & Research

**Overview:**  
Generates blog plans and performs initial research using AI agents and external data sources like sitemaps and categories. This block supports deep research branching logic.

**Nodes Involved:**  
- Blog Planner (Langchain Chain LLM)  
- Check Empty Output (If)  
- Create Drive Folder (Google Drive)  
- Get Planner Output (Set)  
- HTTP Request Get Categories  
- Edit Fields Categories (Set)  
- Aggregate Categories (Aggregate)  
- Get Post Sitemap (HTTP Request)  
- Get XML File (XML Parse)  
- Split Out Links  
- Limit Inbound Links  
- Aggregate internal links  
- Switch Deep Research (Switch)  
- Initial Research (Langchain Agent)  
- Structured Output Parser4 (Langchain Output Parser Structured)  
- Message a model in Perplexity (Perplexity Tool)  

**Node Details:**

- **Blog Planner**  
  - Type: Langchain Chain LLM  
  - Role: Uses AI to generate a content plan or outline for the topic.  
  - Input: Topic data  
  - Output: Planned blog structure and topics.  
  - Failure Modes: AI request timeout, invalid prompts.

- **Check Empty Output**  
  - Type: If  
  - Role: Checks if planning output is empty; decides next steps.  
  - Input: Blog Planner output  
  - Output: Branches to create folders or proceed with planning  

- **Create Drive Folder**  
  - Type: Google Drive  
  - Role: Creates a folder for storing images or files related to the blog post.  
  - Input: Triggered on empty output decision path.  
  - Output: Folder ID for downstream image uploads.

- **Get Planner Output**  
  - Type: Set  
  - Role: Stores or formats output from the planner for further use.  

- **HTTP Request Get Categories**  
  - Type: HTTP Request  
  - Role: Retrieves categories from external source or API for the blog.  
  - Output: Categories JSON.

- **Edit Fields Categories**  
  - Type: Set  
  - Role: Formats or edits category fields for Airtable compatibility.  

- **Aggregate Categories**  
  - Type: Aggregate  
  - Role: Aggregates category data for batch processing.

- **Get Post Sitemap**  
  - Type: HTTP Request  
  - Role: Fetches sitemap XML file from website for internal linking research.

- **Get XML File**  
  - Type: XML Parse  
  - Role: Parses sitemap XML into JSON for further processing.

- **Split Out Links**  
  - Type: SplitOut  
  - Role: Splits list of links from sitemap for individual processing.

- **Limit Inbound Links**  
  - Type: Limit  
  - Role: Restricts number of inbound links processed to avoid overload.

- **Aggregate internal links**  
  - Type: Aggregate  
  - Role: Aggregates filtered internal links for use in research.

- **Switch Deep Research**  
  - Type: Switch  
  - Role: Decides if deep research branch is taken based on configuration or input.

- **Initial Research**  
  - Type: Langchain Agent  
  - Role: Uses AI to perform initial content research based on topic and links.

- **Structured Output Parser4**  
  - Type: Langchain Output Parser Structured  
  - Role: Parses AI output into structured format for further processing.

- **Message a model in Perplexity**  
  - Type: Perplexity Tool  
  - Role: Optionally queries Perplexity AI for additional research data.

---

#### 2.3 Content Generation & Chapter Handling

**Overview:**  
Generates chapters and subchapters content via AI agents, manages batch processing, updates Airtable records, and loops over content pieces for processing and deletion cleanup.

**Nodes Involved:**  
- Airtable Generate Content  
- Airtable Generate Chapters  
- Airtable Select Content  
- Airtable Select Chapters  
- Split Content IDs  
- Split Chapter IDs  
- Loop Over Content  
- Loop Over Chapters  
- Loop Over Content Process  
- Loop Over Chapters Process  
- Split Out Content Process IDs  
- Split Out Chapter Process IDs  
- Combine Into Article  
- Final Article In Markdown  
- Markdown To HTML  
- FInal Article In HTML  
- Structured Output Parser (Langchain)  
- Researcher (Langchain Agent)  
- Copywriter (Langchain Chain LLM)  
- Subchapter Researcher (Langchain Agent)  
- Subchapter Copywriter (Langchain Chain LLM)  
- Split Out Subchapters  
- Loop Over Sub-Chapter  
- Aggregate  
- Aggregate1  
- Code (Custom code nodes for content formatting and parsing)  
- Wait To Save Content  
- Wait To Save Chapters  
- Wait To Update Data  
- Wait To Update Data1  

**Node Details:**

- **Airtable Generate Content / Chapters**  
  - Type: Airtable Read  
  - Role: Retrieves generated content/chapter records for processing.  
  - Input: Triggered post-research/planning.

- **Split Content IDs / Split Chapter IDs**  
  - Type: SplitOut  
  - Role: Splits lists of content or chapter IDs for batch processing.

- **Loop Over Content / Chapters**  
  - Type: SplitInBatches  
  - Role: Processes content or chapters in batches to manage API limits and performance.

- **Combine Into Article / Combine Into Article1**  
  - Type: Code  
  - Role: Custom logic to combine chapter contents into a single article.

- **Final Article In Markdown / Markdown To HTML / FInal Article In HTML**  
  - Types: Set, Markdown, Set  
  - Role: Converts combined Markdown content to HTML suitable for WordPress.

- **Structured Output Parser**  
  - Type: Langchain Output Parser Structured  
  - Role: Parses AI-generated text into defined structured formats.

- **Researcher / Copywriter / Subchapter Researcher / Subchapter Copywriter**  
  - Types: Langchain Agent, Chain LLM  
  - Role: AI nodes that generate or refine content based on research or writing tasks.

- **Split Out Subchapters, Loop Over Sub-Chapter**  
  - Types: SplitOut, SplitInBatches  
  - Role: Handles subchapter level processing similarly to chapters.

- **Aggregate / Aggregate1**  
  - Type: Aggregate  
  - Role: Aggregates processed chunks for final output.

- **Wait Nodes**  
  - Types: Wait (Webhook)  
  - Role: Introduces delays or waits for external confirmations during processing.

- **Code Nodes**  
  - Role: Custom JavaScript for parsing, combining, or formatting content.

---

#### 2.4 Image Processing & Upload

**Overview:**  
Manages featured and chapter image processing including generation via OpenAI, resizing, uploading to Google Drive, HTTP uploads, and updating Airtable with image metadata. Supports both AI-generated and manual images.

**Nodes Involved:**  
- HTTP Request Open AI Image  
- HTTP Request Open AI Image 1  
- Generate Image (Set)  
- Manual Image (Set)  
- Resize featured image  
- Upload featured image to Drive  
- Upload Featured Image (HTTP Request)  
- Upload Featured Image1 (HTTP Request)  
- Set featured image for post  
- Set featured image for post1  
- Airtable Chapter Images  
- Airtable Chapter Images1  
- Airtable Chapter Images2  
- Airtable Chapter Images3  
- Airtable Chapter Images4  
- Airtable Select Chapter Images1  
- Split Out Chapter Image  
- Filter Chapter Image Data  
- Convert Data To File  
- Resize Chapter Image  
- Upload Chapter Image  
- Upload Chapter Image1  
- Upload Chapter Image To Drive  
- Merge Chapter Image Data  
- Update Chapter Image Meta Data  
- Update Featured Image Meta Data  
- If Manual Image / If Manual Image1 (If nodes to branch manual image paths)  

**Node Details:**

- **HTTP Request Open AI Image / HTTP Request Open AI Image 1**  
  - Type: HTTP Request  
  - Role: Requests AI-generated images from OpenAI or similar service.

- **Generate Image / Manual Image**  
  - Type: Set  
  - Role: Sets parameters for image type - either AI-generated or manual.

- **Resize featured image / Resize Chapter Image**  
  - Type: Edit Image  
  - Role: Resizes images to required dimensions before upload.

- **Upload featured image to Drive / Upload Chapter Image To Drive**  
  - Type: Google Drive  
  - Role: Uploads resized images to Google Drive folders.

- **Upload Featured Image / Upload Chapter Image / HTTP Request nodes**  
  - Type: HTTP Request  
  - Role: Uploads images to external servers or WordPress endpoints.

- **Set featured image for post / Set featured image for post1**  
  - Type: HTTP Request  
  - Role: Associates uploaded images as featured images on WordPress posts.

- **Airtable Chapter Images and Select Chapter Images nodes**  
  - Type: Airtable Read  
  - Role: Retrieves image metadata from Airtable for chapters.

- **Split Out Chapter Image / Filter Chapter Image Data / Filter Feature Image Data**  
  - Types: SplitOut, Filter  
  - Role: Splits and filters images based on metadata and status.

- **Convert Data To File / Convert to File**  
  - Type: Convert To File  
  - Role: Converts image data from base64 or other forms to file blobs for upload.

- **Merge Chapter Image Data**  
  - Type: Merge  
  - Role: Merges image upload results with other chapter data.

- **Update Chapter Image Meta Data / Update Featured Image Meta Data**  
  - Type: HTTP Request  
  - Role: Updates metadata on WordPress or external platforms post-upload.

- **If Manual Image / If Manual Image1**  
  - Type: If  
  - Role: Branches workflow based on whether image is manual or AI-generated.

---

#### 2.5 Content Finalization & Publication

**Overview:**  
Finalizes content by merging article and feature images, updates Airtable post records, and publishes the post on WordPress with proper metadata and featured images.

**Nodes Involved:**  
- Merge Article And Feature Image  
- Airtable Finalize Post  
- Airtable Finalize Post1, Post2, Post3, Post4  
- Airtable Finalize Post - Update Status  
- Airtable Finalize Post - Delete Record  
- Post on Wordpress  
- Set excerpt  
- Wait To Save Post  
- Wait To Backup Post  
- Airtable Backup Post  
- Wait To Delete Post  
- Wait To Delete Post1  

**Node Details:**

- **Merge Article And Feature Image**  
  - Type: Merge  
  - Role: Combines final article HTML content with feature image data before publishing.

- **Airtable Finalize Post nodes**  
  - Type: Airtable Update/Delete  
  - Role: Updates or deletes post records in Airtable to reflect publishing status.

- **Post on Wordpress**  
  - Type: Wordpress node  
  - Role: Publishes the finalized post to WordPress site.

- **Set excerpt**  
  - Type: HTTP Request  
  - Role: Updates post excerpt metadata on WordPress.

- **Wait To Save Post / Wait To Backup Post / Wait To Delete Post**  
  - Type: Wait (Webhook)  
  - Role: Synchronizes post saving, backup, and cleanup operations.

- **Airtable Backup Post**  
  - Type: Airtable Update  
  - Role: Creates backup copies of published post data in Airtable.

- **Wait To Delete Post / Wait To Delete Post1**  
  - Type: Wait (Webhook)  
  - Role: Delays deletion of post records to ensure safe cleanup.

---

#### 2.6 Cleanup & Status Updates

**Overview:**  
Manages deletion of old or processed content, chapters, and generated content records. Updates statuses in Airtable to reflect current progress and cleans up temporary data.

**Nodes Involved:**  
- Airtable Select Content - Delete Record  
- Airtable Select Chapters - Delete Record  
- Airtable Generate Content - Delete Record  
- Airtable Generate Chapters - Delete Record  
- Wait To Delete Content Process  
- Wait To Delete Chapters Process  
- Wait To Delete Generate Content  
- Wait To Delete Chapter  
- Wait To Delete Post  
- Wait To Delete Post1  
- Airtable Select Chapters - Create Record  
- Wait To Create Record  
- Check Empty Output (If)  
- Check Status And Confirmation  
- Filter Post Status And Confirmation  
- Filter Chapter Images  
- Filter Feature Image Data  
- Filter Status  

**Node Details:**

- **Delete Record Nodes**  
  - Type: Airtable Delete  
  - Role: Removes temporary or processed records from Airtable tables.

- **Wait To Delete... nodes**  
  - Type: Wait (Webhook)  
  - Role: Introduces delay to ensure asynchronous operations complete before deletions.

- **Airtable Select Chapters - Create Record**  
  - Type: Airtable Create  
  - Role: Creates new chapter records after processing.

- **Check & Filter Nodes**  
  - Role: Validates record statuses and filters to maintain workflow correctness.

---

### 3. Summary Table

| Node Name                          | Node Type                    | Functional Role                    | Input Node(s)                           | Output Node(s)                          | Sticky Note              |
|-----------------------------------|------------------------------|----------------------------------|---------------------------------------|---------------------------------------|--------------------------|
| Create New Topic                  | Airtable Trigger             | Trigger on new topic             | -                                     | Airtable Get Topic                     |                          |
| Airtable Get Topic                | Airtable                     | Retrieve topic records           | Create New Topic                      | Split Out All Records                  |                          |
| Split Out All Records             | SplitOut                    | Split array of records           | Airtable Get Topic                    | Filter Status                         |                          |
| Filter Status                    | Filter                      | Filter topics by status          | Split Out All Records                 | Limit To First Item                   |                          |
| Limit To First Item               | Limit                       | Limit processing to first item   | Filter Status                        | Check Status & Confirmation           |                          |
| Check Status & Confirmation      | If                          | Check confirmation flag          | Limit To First Item                   | Confirm Topic / End                   |                          |
| Confirm Topic                   | Airtable                     | Update topic confirmation        | Check Status & Confirmation (True)  | Wait To Update New Topic              |                          |
| Wait To Update New Topic         | Wait (Webhook)               | Wait for external confirmation   | Confirm Topic                       | Multiple Airtable nodes               |                          |
| Settings                       | Code                        | Configuration storage            | -                                   | Check inputs                        |                          |
| Blog Planner                   | Langchain Chain LLM          | AI content planning              | Check Empty Output (False)           | Check Empty Output                    |                          |
| Check Empty Output              | If                          | Check if planning output empty   | Blog Planner                       | Create Drive Folder / Blog Planner    |                          |
| Create Drive Folder             | Google Drive                | Create drive folder for images   | Check Empty Output (True)            | Get Planner Output                   |                          |
| Get Planner Output             | Set                         | Store planner output             | Create Drive Folder                 | Split Out chapters / HTTP Request Open AI Image 1 |                          |
| HTTP Request Get Categories    | HTTP Request                | Retrieve categories              | Settings2                         | Edit Fields Categories               |                          |
| Edit Fields Categories          | Set                         | Format categories                | HTTP Request Get Categories        | Aggregate Categories                 |                          |
| Aggregate Categories            | Aggregate                   | Aggregate categories             | Edit Fields Categories             | Airtable Select Chapters2            |                          |
| Get Post Sitemap               | HTTP Request                | Fetch sitemap XML                | Settings1                         | Get XML File                       |                          |
| Get XML File                  | XML                         | Parse sitemap XML                | Get Post Sitemap                  | Split Out Links                    |                          |
| Split Out Links               | SplitOut                    | Split sitemap links              | Get XML File                     | Limit Inbound Links                 |                          |
| Limit Inbound Links          | Limit                       | Limit inbound link count         | Split Out Links                  | Aggregate internal links           |                          |
| Aggregate internal links      | Aggregate                   | Aggregate links                 | Limit Inbound Links             | Airtable Chapters1                |                          |
| Switch Deep Research          | Switch                      | Branch for deep research        | Parse Chapters                  | Split Out / Split Out1             |                          |
| Initial Research             | Langchain Agent             | AI initial research             | Check inputs                  | Wait For AI APIs                  |                          |
| Structured Output Parser4    | Langchain Output Parser Structured | Parse AI output                 | Initial Research             | Message a model in Perplexity      |                          |
| Message a model in Perplexity | Perplexity Tool             | Optional external AI research   | Structured Output Parser4    | Initial Research continuation      |                          |
| Airtable Generate Content      | Airtable                    | Get generated content           | Airtable Select Content          | Split Content IDs                |                          |
| Airtable Generate Chapters     | Airtable                    | Get generated chapters          | Airtable Select Chapters         | Split Chapter IDs               |                          |
| Split Content IDs              | SplitOut                   | Split content IDs               | Airtable Generate Content       | Loop Over Content              |                          |
| Split Chapter IDs             | SplitOut                   | Split chapter IDs               | Airtable Generate Chapters      | Loop Over Chapters             |                          |
| Loop Over Content             | SplitInBatches             | Batch process content           | Split Content IDs              | Airtable Generate Content - Delete Record |                          |
| Loop Over Chapters            | SplitInBatches             | Batch process chapters          | Split Chapter IDs              | Airtable Generate Chapters - Delete Record |                          |
| Combine Into Article          | Code                       | Combine chapter content         | Loop Over Chapters2            | Final Article In Markdown       |                          |
| Final Article In Markdown     | Set                        | Store combined markdown         | Combine Into Article           | Markdown To HTML              |                          |
| Markdown To HTML              | Markdown                   | Convert Markdown to HTML        | Final Article In Markdown      | FInal Article In HTML          |                          |
| FInal Article In HTML         | Set                        | Store final HTML                | Markdown To HTML              | Merge Article And Feature Image |                          |
| Structured Output Parser      | Langchain Output Parser Structured | Parse AI writing outputs       | Copywriter                   | Airtable Generate Content1       |                          |
| Researcher                   | Langchain Agent             | AI research agent               | OpenAI Chat Model1             | Copywriter                     |                          |
| Copywriter                   | Langchain Chain LLM          | AI writing agent               | Researcher                    | Airtable Generate Content1       |                          |
| Subchapter Researcher        | Langchain Agent             | AI subchapter research         | OpenAI Chat Model2             | Wait4                         |                          |
| Subchapter Copywriter        | Langchain Chain LLM          | AI subchapter writing          | OpenAI Chat Model3             | Wait2                         |                          |
| Split Out Subchapters        | SplitOut                   | Split subchapter records       | Loop Over Chapter             | Loop Over Sub-Chapter          |                          |
| Loop Over Sub-Chapter        | SplitInBatches             | Batch process subchapters      | Split Out Subchapters          | Subchapter Researcher / Subchapter Copywriter |                          |
| Aggregate                   | Aggregate                   | Aggregate subchapter data      | Loop Over Chapter             | Subchapters Output            |                          |
| Subchapters Output           | Set                        | Output combined subchapter data | Aggregate                      | Code                         |                          |
| Code                       | Code                       | Custom content formatting      | Subchapters Output            | Airtable Generate Content2     |                          |
| Airtable Select Content      | Airtable                    | Select content records          | Wait To Update New Topic       | Split Content Process IDs      |                          |
| Airtable Select Chapters     | Airtable                    | Select chapter records          | Wait To Update New Topic       | Split Chapter Process IDs      |                          |
| Airtable Select Content - Delete Record | Airtable          | Delete old content records     | Loop Over Content Process      | Wait To Delete Content Process |                          |
| Airtable Select Chapters - Delete Record | Airtable          | Delete old chapter records     | Loop Over Chapters Process     | Wait To Delete Chapters Process |                          |
| Airtable Generate Content - Delete Record | Airtable          | Delete generated content       | Loop Over Content             | Wait To Delete Generate Content |                          |
| Airtable Generate Chapters - Delete Record | Airtable          | Delete generated chapters      | Loop Over Chapters             | Wait To Delete Chapter         |                          |
| Wait To Delete Content Process | Wait (Webhook)            | Wait before content deletion   | Airtable Select Content - Delete Record | Loop Over Content Process      |                          |
| Wait To Delete Chapters Process | Wait (Webhook)           | Wait before chapters deletion  | Airtable Select Chapters - Delete Record | Loop Over Chapters Process     |                          |
| Wait To Delete Generate Content | Wait (Webhook)            | Wait before generated content deletion | Airtable Generate Content - Delete Record | Loop Over Content             |                          |
| Wait To Delete Chapter         | Wait (Webhook)             | Wait before chapter deletion   | Airtable Generate Chapters - Delete Record | Loop Over Chapters             |                          |
| Airtable Finalize Post        | Airtable                    | Update post status             | Split Post IDs                | Split Out Records              |                          |
| Split Post IDs                | SplitOut                   | Split post IDs                | Airtable Finalize Post       | Loop Over Records             |                          |
| Loop Over Records             | SplitInBatches             | Batch process post finalization | Split Post IDs               | Airtable Finalize Post - Delete Record |                          |
| Airtable Finalize Post - Delete Record | Airtable             | Delete post records           | Loop Over Records             | Wait To Delete Post            |                          |
| Wait To Delete Post           | Wait (Webhook)             | Wait before deleting posts     | Airtable Finalize Post - Delete Record | Loop Over Records              |                          |
| Post on Wordpress             | Wordpress                  | Publish post on WordPress      | Limit To First Item3           | If Manual Image1               |                          |
| Set excerpt                  | HTTP Request               | Update post excerpt metadata   | Post on Wordpress             | Feature Image                 |                          |
| Feature Image                | Set                        | Set feature image data         | Set excerpt                  | Airtable Backup Post           |                          |
| Airtable Backup Post          | Airtable                    | Backup post data               | Feature Image                | Wait To Backup Post            |                          |
| Wait To Backup Post           | Wait (Webhook)             | Wait before backing up         | Airtable Backup Post          | Airtable Finalize Post - Update Status |                          |
| Airtable Finalize Post - Update Status | Airtable             | Update post status in Airtable | Wait To Backup Post           | -                             |                          |
| Resize featured image         | Edit Image                 | Resize feature image           | Convert to File               | Upload featured image to Drive |                          |
| Upload featured image to Drive | Google Drive               | Upload feature image           | Resize featured image         | Merge3                       |                          |
| Merge3                       | Merge                      | Merge feature image data       | Upload featured image to Drive | Upload Featured Image         |                          |
| Upload Featured Image         | HTTP Request               | Upload feature image to WordPress | Merge3                    | Update Featured Image Meta Data |                          |
| Update Featured Image Meta Data | HTTP Request              | Update feature image metadata  | Upload Featured Image         | Merge Article And Feature Image |                          |
| Merge Article And Feature Image | Merge                    | Combine article and feature image | FInal Article In HTML       | Airtable Finalize Post1        |                          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Airtable Triggers**  
   - Create "Create New Topic" Airtable Trigger node to start workflow on new topic entries.  
   - Create "Airtable Finalize Post Trigger" for post finalization events.  
   - Create other Airtable triggers as needed for content, chapters, and images.

2. **Retrieve and Filter Topics**  
   - Add "Airtable Get Topic" node to fetch topic data.  
   - Add "Split Out All Records" to split topics.  
   - Add "Filter Status" to filter topics by processing status.  
   - Add "Limit To First Item" to limit to the first topic.  
   - Add "Check Status & Confirmation" (If node) to check if topic is confirmed.

3. **Confirm Topic and Wait**  
   - Add "Confirm Topic" Airtable node to update confirmation status.  
   - Add "Wait To Update New Topic" wait node (Webhook) to delay further processing.

4. **Settings Node**  
   - Add a "Code" node to store and manage configurable settings.

5. **Content Planning & Research**  
   - Add "Blog Planner" Langchain Chain LLM node for AI content planning.  
   - Add "Check Empty Output" If node to branch logic based on planner output.  
   - Add "Create Drive Folder" Google Drive node for image storage.  
   - Add "HTTP Request Get Categories" node to fetch categories.  
   - Add "Edit Fields Categories" and "Aggregate Categories" nodes for category formatting and aggregation.  
   - Add "Get Post Sitemap" HTTP Request node and "Get XML File" XML node to fetch and parse sitemap.  
   - Add "Split Out Links," "Limit Inbound Links," and "Aggregate internal links" for link management.  
   - Add "Switch Deep Research" node to branch deep research logic.  
   - Add "Initial Research" Langchain Agent node for AI research.  
   - Add "Structured Output Parser" nodes to parse AI outputs.  
   - Optionally add "Message a model in Perplexity" for additional AI research.

6. **Content Generation & Chapter Handling**  
   - Add Airtable nodes to "Generate Content" and "Generate Chapters."  
   - Add SplitOut nodes to split content and chapter IDs.  
   - Add SplitInBatches nodes to loop over content and chapters.  
   - Add Langchain Agent and Chain LLM nodes for "Researcher," "Copywriter," "Subchapter Researcher," and "Subchapter Copywriter."  
   - Add nodes to combine content into articles, convert Markdown to HTML, and prepare final HTML.  
   - Add wait nodes to synchronize saving and updating data.

7. **Image Processing & Upload**  
   - Add HTTP Request nodes to request AI-generated images.  
   - Use Set nodes to mark manual or AI images.  
   - Use Edit Image nodes to resize images.  
   - Add Google Drive nodes to upload images.  
   - Add HTTP Request nodes to upload images to WordPress or external services.  
   - Add Airtable nodes to select and update image metadata.  
   - Include If nodes to branch manual image processing paths.

8. **Content Finalization & Publication**  
   - Add Merge node to combine article content and featured images.  
   - Add Airtable Finalize Post nodes to update status and records.  
   - Add Wordpress node to publish posts.  
   - Add HTTP Request nodes to update excerpts and featured images metadata.  
   - Add wait nodes for synchronization of saving, backup, and deletion.

9. **Cleanup & Status Updates**  
   - Add Airtable Delete nodes to remove processed content, chapters, and posts.  
   - Add Wait nodes to delay deletions ensuring all processes complete.  
   - Add Filter and If nodes to manage status validation before deletion.  
   - Add Airtable Create nodes if new chapters need to be created after cleanup.

10. **Workflow Connections**  
    - Connect nodes according to logical flow: triggers → data retrieval → filtering → AI processing → data storage → image processing → content finalization → publication → cleanup.  
    - Use SplitInBatches and SplitOut nodes to handle arrays and batch operations.  
    - Use If and Switch nodes to branch based on conditions like status or manual image flags.  
    - Use Wait nodes for webhook-based delays to synchronize asynchronous operations.

11. **Credential Setup**  
    - Configure Airtable credentials with appropriate API keys.  
    - Configure OpenAI credentials for Langchain nodes.  
    - Configure Google Drive OAuth2 credentials for image storage.  
    - Configure WordPress credentials for post publication.  
    - Configure Perplexity API if Perplexity nodes are used.

---

### 5. General Notes & Resources

| Note Content                                                                                       | Context or Link                                                                                  |
|--------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Workflow tagged as "Template" for reuse and modification by advanced users.                      | n8n template tagging system                                                                     |
| Uses Langchain nodes for AI agents, requiring OpenAI API keys and model configuration.           | https://docs.n8n.io/integrations/ai/langchain/                                                 |
| Incorporates Perplexity AI tool nodes for optional research augmentation.                         | Perplexity AI integration documentation                                                        |
| Handles image processing via Google Drive API and WordPress HTTP endpoints with OAuth2.          | Google Drive API docs: https://developers.google.com/drive/api<br>WordPress REST API docs       |
| Employs webhook-based Wait nodes for asynchronous flow control to synchronize external events.   | n8n Webhook node and wait node documentation                                                   |
| Utilizes Airtable extensively as CMS; ensure API rate limits are respected during batch loops.   | Airtable API docs: https://airtable.com/api                                                      |
| Extensive use of batch processing nodes (SplitInBatches) to handle large datasets efficiently.   | n8n SplitInBatches node documentation                                                          |
| Custom code nodes use JavaScript for data transformation; maintain code quality and test scripts.| n8n Code node reference                                                                        |

---

This structured documentation captures all aspects of the workflow, enabling users and automation agents to understand, reproduce, and maintain the WordPress blog automation process with human-in-the-loop and AI research enhancements.