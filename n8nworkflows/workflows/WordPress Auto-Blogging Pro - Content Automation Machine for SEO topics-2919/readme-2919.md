WordPress Auto-Blogging Pro - Content Automation Machine for SEO topics

https://n8nworkflows.xyz/workflows/wordpress-auto-blogging-pro---content-automation-machine-for-seo-topics-2919


# WordPress Auto-Blogging Pro - Content Automation Machine for SEO topics

### 1. Workflow Overview

This workflow, **WordPress Auto-Blogging Pro - Content Automation Machine for SEO topics**, is designed to fully automate the creation, enrichment, publication, and backup of SEO-optimized blog posts on a WordPress website. It targets content creators, SEO specialists, and digital marketers who want to streamline content production with AI assistance, image generation, and internal linking strategies while ensuring content safety via Google Drive backups.

The workflow logically divides into the following blocks:

- **1.1 Input Reception and Validation:** Triggered by new rows in Google Sheets, it validates and extracts input parameters for blog post generation.
- **1.2 Blog Post Structure and Title Generation:** Uses AI to create the blog post title and chapter structure.
- **1.3 Internal Link Collection:** Fetches the website sitemap, extracts internal links, and limits them for SEO embedding.
- **1.4 Chapter Content Generation:** Iteratively generates text content for each chapter using AI.
- **1.5 Image Generation and Processing:** Creates unique images per chapter and a featured image, resizes, uploads, and updates metadata.
- **1.6 Article Assembly and Formatting:** Combines chapters, converts markdown to HTML, and prepares the final article.
- **1.7 Publishing and Backup:** Publishes the article on WordPress, sets featured images, and saves all content and images to Google Drive.
- **1.8 Post-Publication Metadata Updates:** Updates excerpts and image metadata on WordPress for SEO optimization.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Validation

- **Overview:** This block triggers the workflow upon new data entry in Google Sheets, extracts the latest topic row, and validates the inputs to ensure all required parameters are present before proceeding.
- **Nodes Involved:**  
  - Google Sheets Trigger  
  - Get the latest topic (Limit)  
  - Settings (Set)  
  - Check inputs (If)  
- **Node Details:**

  - **Google Sheets Trigger**  
    - Type: Trigger  
    - Role: Starts workflow on new row addition in Google Sheets.  
    - Configuration: Monitors a specific Google Sheet for new rows.  
    - Inputs: None (trigger)  
    - Outputs: Row data with fields like Topic, Style, Word Count, etc.  
    - Edge cases: Missing or malformed sheet data; Google Sheets API auth errors.

  - **Get the latest topic**  
    - Type: Limit  
    - Role: Limits to the most recent row to process one topic at a time.  
    - Configuration: Limit set to 1.  
    - Inputs: Data from Google Sheets Trigger.  
    - Outputs: Single latest topic row.  
    - Edge cases: Empty sheet or no new rows.

  - **Settings**  
    - Type: Set  
    - Role: Prepares or normalizes input parameters for downstream nodes.  
    - Configuration: Sets variables for topic, chapters, style, etc.  
    - Inputs: Latest topic data.  
    - Outputs: Structured parameters.  
    - Edge cases: Missing or invalid parameters.

  - **Check inputs**  
    - Type: If  
    - Role: Validates presence and correctness of required inputs before continuing.  
    - Configuration: Checks key fields like Topic, Word Count, Chapters.  
    - Inputs: Settings node output.  
    - Outputs: Branches workflow on validation success or failure.  
    - Edge cases: Missing required fields, invalid numeric values.

---

#### 2.2 Blog Post Structure and Title Generation

- **Overview:** Generates the blog post title and chapter structure using an AI language model with structured output parsing.
- **Nodes Involved:**  
  - OpenRouter Chat Model  
  - Create post title and structure (Chain LLM)  
  - Structured Output Parser2  
  - Create Drive folder  
  - Get output (Set)  
- **Node Details:**

  - **OpenRouter Chat Model**  
    - Type: AI Language Model (OpenRouter)  
    - Role: Provides AI completions for title and structure generation.  
    - Configuration: Uses OpenRouter credentials, model selection configurable.  
    - Inputs: Validated input parameters (topic, style, etc.)  
    - Outputs: Raw AI response.  
    - Edge cases: API rate limits, auth failures, model errors.

  - **Create post title and structure**  
    - Type: Chain LLM  
    - Role: Processes AI response to generate structured post title and chapter outline.  
    - Configuration: Uses prompt templates with input variables.  
    - Inputs: AI model output.  
    - Outputs: Structured title and chapter list.  
    - Edge cases: Parsing errors, incomplete AI output.

  - **Structured Output Parser2**  
    - Type: Output Parser  
    - Role: Parses AI output into JSON or structured format for further processing.  
    - Inputs: AI response from Create post title and structure.  
    - Outputs: Parsed structured data.  
    - Edge cases: Parsing failures due to unexpected AI output format.

  - **Create Drive folder**  
    - Type: Google Drive  
    - Role: Creates a folder named after the blog post title for saving content and images.  
    - Inputs: Parsed post title.  
    - Outputs: Folder ID and metadata.  
    - Edge cases: Google Drive API errors, permission issues.

  - **Get output**  
    - Type: Set  
    - Role: Prepares output variables for next steps (chapter splitting, image generation).  
    - Inputs: Folder info and structured post data.  
    - Outputs: Variables for downstream nodes.  
    - Edge cases: Missing folder ID or post data.

---

#### 2.3 Internal Link Collection

- **Overview:** Retrieves the website sitemap XML, extracts internal links, and limits the number of links to embed for SEO enhancement.
- **Nodes Involved:**  
  - Get post sitemap (HTTP Request)  
  - Get XML file (XML Parser)  
  - Split out links (Split Out)  
  - Limit1 (Limit)  
  - Aggregate internal links (Aggregate)  
- **Node Details:**

  - **Get post sitemap**  
    - Type: HTTP Request  
    - Role: Downloads the sitemap XML from the target website.  
    - Configuration: URL parameterized by input Website field.  
    - Inputs: Website URL.  
    - Outputs: Sitemap XML content.  
    - Edge cases: HTTP errors, invalid sitemap URL, network issues.

  - **Get XML file**  
    - Type: XML Parser  
    - Role: Parses sitemap XML to extract URLs.  
    - Inputs: Sitemap XML content.  
    - Outputs: Parsed URL list.  
    - Edge cases: Malformed XML, parsing errors.

  - **Split out links**  
    - Type: Split Out  
    - Role: Splits parsed URLs into individual items for processing.  
    - Inputs: Parsed URL list.  
    - Outputs: Individual URL items.  
    - Edge cases: Empty URL list.

  - **Limit1**  
    - Type: Limit  
    - Role: Restricts number of internal links to 20 by default to avoid over-embedding.  
    - Inputs: Individual URLs.  
    - Outputs: Limited URL list.  
    - Edge cases: Less than 20 links available.

  - **Aggregate internal links**  
    - Type: Aggregate  
    - Role: Collects limited internal links into a single list for embedding.  
    - Inputs: Limited URL list.  
    - Outputs: Aggregated internal links array.  
    - Edge cases: Empty aggregation if no links.

---

#### 2.4 Chapter Content Generation

- **Overview:** Iteratively generates text content for each chapter using AI, embedding internal links and applying writing style and SEO parameters.
- **Nodes Involved:**  
  - Split out chapters (Split Out)  
  - Loop Over Items (Split In Batches)  
  - OpenRouter Chat Model1  
  - Write chapter text (Chain LLM)  
  - Structured Output Parser  
  - Loop Over Items1 (Split In Batches)  
  - Get title chapter and content (Set)  
  - Wait (Wait) [disabled]  
- **Node Details:**

  - **Split out chapters**  
    - Type: Split Out  
    - Role: Splits chapter structure into individual chapters for processing.  
    - Inputs: Structured chapter list.  
    - Outputs: Single chapter data items.  
    - Edge cases: Empty chapter list.

  - **Loop Over Items**  
    - Type: Split In Batches  
    - Role: Processes chapters in batches to manage API rate limits and workflow performance.  
    - Inputs: Chapter items.  
    - Outputs: Batched chapter items.  
    - Edge cases: Batch size misconfiguration.

  - **OpenRouter Chat Model1**  
    - Type: AI Language Model (OpenRouter)  
    - Role: Generates chapter text based on chapter title, style, SEO keywords, and internal links.  
    - Inputs: Chapter data, internal links, style parameters.  
    - Outputs: Raw AI-generated chapter text.  
    - Edge cases: API errors, rate limits.

  - **Write chapter text**  
    - Type: Chain LLM  
    - Role: Processes AI output to generate clean chapter text.  
    - Inputs: AI model output.  
    - Outputs: Chapter text content.  
    - Edge cases: Parsing or generation errors.

  - **Structured Output Parser**  
    - Type: Output Parser  
    - Role: Parses AI output into structured chapter content.  
    - Inputs: AI output from Write chapter text.  
    - Outputs: Parsed chapter text.  
    - Edge cases: Parsing failures.

  - **Loop Over Items1**  
    - Type: Split In Batches  
    - Role: Further batch processing for chapter content refinement or image generation.  
    - Inputs: Chapter text items.  
    - Outputs: Batched chapter content.  
    - Edge cases: Batch size issues.

  - **Get title chapter and content**  
    - Type: Set  
    - Role: Prepares chapter title and content for merging and image generation.  
    - Inputs: Parsed chapter content.  
    - Outputs: Structured chapter data.  
    - Edge cases: Missing content or title.

  - **Wait** (disabled)  
    - Type: Wait  
    - Role: Intended to delay processing to avoid rate limits (currently disabled).  
    - Inputs: Chapter processing flow.  
    - Outputs: Delayed flow.  
    - Edge cases: If enabled, must be tuned to avoid excessive delays.

---

#### 2.5 Image Generation and Processing

- **Overview:** Generates unique images for each chapter and a featured image for the article, resizes them, uploads to Google Drive and WordPress, and updates metadata.
- **Nodes Involved:**  
  - Generate chapter image (OpenAI)  
  - Resize Image (Edit Image)  
  - Upload chapter images (HTTP Request)  
  - Update image meta data (HTTP Request)  
  - Upload chapter images to Drive (Google Drive)  
  - Generate featured image (OpenAI)  
  - Resize featured image (Edit Image)  
  - Upload featured image to Drive (Google Drive)  
  - Upload featured image (HTTP Request)  
  - Update featured image meta data (HTTP Request)  
  - Loop Over Items1 (Split In Batches)  
  - Merge1, Merge2, Add featured image (Merge)  
- **Node Details:**

  - **Generate chapter image**  
    - Type: OpenAI (Langchain)  
    - Role: Creates AI-generated images based on chapter content or prompts.  
    - Inputs: Chapter titles/content.  
    - Outputs: Image URLs or base64 data.  
    - Edge cases: API limits, image generation failures.

  - **Resize Image**  
    - Type: Edit Image  
    - Role: Resizes chapter images to required dimensions for WordPress and SEO.  
    - Inputs: Generated images.  
    - Outputs: Resized images.  
    - Edge cases: Image processing errors.

  - **Upload chapter images**  
    - Type: HTTP Request  
    - Role: Uploads images to WordPress media library via REST API.  
    - Inputs: Resized images.  
    - Outputs: Media IDs and URLs.  
    - Edge cases: Authentication errors, upload failures.

  - **Update image meta data**  
    - Type: HTTP Request  
    - Role: Updates image metadata (alt text, captions) on WordPress for SEO.  
    - Inputs: Media IDs, metadata.  
    - Outputs: Confirmation of update.  
    - Edge cases: API errors.

  - **Upload chapter images to Drive**  
    - Type: Google Drive  
    - Role: Saves chapter images to the Google Drive folder created earlier.  
    - Inputs: Image files.  
    - Outputs: Drive file IDs.  
    - Edge cases: Permission or quota issues.

  - **Generate featured image**  
    - Type: OpenAI (Langchain)  
    - Role: Creates a unique featured image for the entire article.  
    - Inputs: Article title or summary.  
    - Outputs: Image data.  
    - Edge cases: Same as chapter image generation.

  - **Resize featured image**  
    - Type: Edit Image  
    - Role: Resizes the featured image to WordPress requirements.  
    - Inputs: Generated featured image.  
    - Outputs: Resized image.  
    - Edge cases: Processing errors.

  - **Upload featured image to Drive**  
    - Type: Google Drive  
    - Role: Saves featured image to Google Drive folder.  
    - Inputs: Resized featured image.  
    - Outputs: Drive file ID.  
    - Edge cases: Drive API errors.

  - **Upload featured image**  
    - Type: HTTP Request  
    - Role: Uploads featured image to WordPress media library.  
    - Inputs: Resized image.  
    - Outputs: Media ID.  
    - Edge cases: Auth or upload errors.

  - **Update featured image meta data**  
    - Type: HTTP Request  
    - Role: Updates metadata for the featured image on WordPress.  
    - Inputs: Media ID and metadata.  
    - Outputs: Confirmation.  
    - Edge cases: API failures.

  - **Loop Over Items1, Merge1, Merge2, Add featured image**  
    - Type: Split In Batches and Merge  
    - Role: Manage batch processing of images and merge results for final article assembly.  
    - Edge cases: Synchronization issues if batches incomplete.

---

#### 2.6 Article Assembly and Formatting

- **Overview:** Merges chapter titles, texts, and images into a complete article, converts markdown to HTML, and prepares the final content for publishing.
- **Nodes Involved:**  
  - Merge chapters title and text (Merge)  
  - Combine into article (Code)  
  - Final article in Markdown (Set)  
  - Markdown to HTML (Markdown)  
  - FInal article in HTML (Set)  
  - Create Doc (Google Docs)  
  - Save texts to Doc (Google Docs)  
- **Node Details:**

  - **Merge chapters title and text**  
    - Type: Merge  
    - Role: Combines individual chapter titles and texts into a unified structure.  
    - Inputs: Chapter data and images.  
    - Outputs: Combined article content.  
    - Edge cases: Missing chapters or content.

  - **Combine into article**  
    - Type: Code  
    - Role: Custom JavaScript to assemble markdown article with embedded images and internal links.  
    - Inputs: Merged chapter data.  
    - Outputs: Complete markdown article.  
    - Edge cases: Code errors, malformed markdown.

  - **Final article in Markdown**  
    - Type: Set  
    - Role: Stores the final markdown content for conversion.  
    - Inputs: Combined markdown article.  
    - Outputs: Markdown text.  
    - Edge cases: Empty content.

  - **Markdown to HTML**  
    - Type: Markdown  
    - Role: Converts markdown article to HTML for WordPress publishing.  
    - Inputs: Markdown text.  
    - Outputs: HTML content.  
    - Edge cases: Conversion errors.

  - **FInal article in HTML**  
    - Type: Set  
    - Role: Holds final HTML content for downstream publishing.  
    - Inputs: Converted HTML.  
    - Outputs: HTML content.  
    - Edge cases: Empty or invalid HTML.

  - **Create Doc**  
    - Type: Google Docs  
    - Role: Creates a new Google Doc to store the article content as backup.  
    - Inputs: Article title and content.  
    - Outputs: Doc ID.  
    - Edge cases: Google Docs API errors.

  - **Save texts to Doc**  
    - Type: Google Docs  
    - Role: Saves the article text into the created Google Doc.  
    - Inputs: Doc ID and content.  
    - Outputs: Confirmation.  
    - Edge cases: Write permission errors.

---

#### 2.7 Publishing and Backup

- **Overview:** Publishes the article on WordPress, sets the featured image, and saves all content and images to Google Drive.
- **Nodes Involved:**  
  - Post on Wordpress (WordPress)  
  - Add featured image (Merge)  
  - Upload featured image (HTTP Request)  
  - Set featured image for post (HTTP Request)  
  - Set excerpt (HTTP Request)  
  - Edit Fields (Set)  
- **Node Details:**

  - **Post on Wordpress**  
    - Type: WordPress  
    - Role: Publishes the blog post with title, content, and category.  
    - Inputs: Final HTML article, category ID, and metadata.  
    - Outputs: Post ID and URL.  
    - Edge cases: Authentication errors, publishing failures.

  - **Add featured image**  
    - Type: Merge  
    - Role: Combines post data with featured image info for final upload.  
    - Inputs: Post data and featured image upload response.  
    - Outputs: Combined data for metadata update.  
    - Edge cases: Missing image or post data.

  - **Upload featured image**  
    - Type: HTTP Request  
    - Role: Uploads featured image to WordPress media library.  
    - Inputs: Resized featured image.  
    - Outputs: Media ID.  
    - Edge cases: API errors.

  - **Set featured image for post**  
    - Type: HTTP Request  
    - Role: Assigns the uploaded featured image to the published post.  
    - Inputs: Post ID and media ID.  
    - Outputs: Confirmation.  
    - Edge cases: API errors.

  - **Set excerpt**  
    - Type: HTTP Request  
    - Role: Sets the post excerpt (summary) for SEO and display purposes.  
    - Inputs: Post ID and excerpt text.  
    - Outputs: Confirmation.  
    - Edge cases: API failures.

  - **Edit Fields**  
    - Type: Set  
    - Role: Final adjustments or cleanup of post fields after publishing.  
    - Inputs: Post metadata.  
    - Outputs: Updated post data.  
    - Edge cases: Missing fields.

---

#### 2.8 Post-Publication Metadata Updates

- **Overview:** Updates metadata for images and featured image on WordPress to improve SEO and accessibility.
- **Nodes Involved:**  
  - Update image meta data (HTTP Request)  
  - Update featured image meta data (HTTP Request)  
- **Node Details:**

  - **Update image meta data**  
    - Type: HTTP Request  
    - Role: Updates alt text, captions, and other metadata for chapter images on WordPress.  
    - Inputs: Media IDs and metadata.  
    - Outputs: Confirmation.  
    - Edge cases: API errors.

  - **Update featured image meta data**  
    - Type: HTTP Request  
    - Role: Updates metadata for the featured image on WordPress.  
    - Inputs: Featured image media ID and metadata.  
    - Outputs: Confirmation.  
    - Edge cases: API failures.

---

### 3. Summary Table

| Node Name                  | Node Type                          | Functional Role                               | Input Node(s)                       | Output Node(s)                    | Sticky Note                                  |
|----------------------------|----------------------------------|-----------------------------------------------|-----------------------------------|---------------------------------|----------------------------------------------|
| Google Sheets Trigger       | Trigger                          | Starts workflow on new Google Sheets row      | None                              | Get the latest topic             |                                              |
| Get the latest topic        | Limit                            | Limits to latest row for processing            | Google Sheets Trigger             | Settings                        |                                              |
| Settings                   | Set                              | Prepares input parameters                      | Get the latest topic              | Check inputs                   |                                              |
| Check inputs               | If                               | Validates required inputs                       | Settings                        | Create post title and structure, Get post sitemap |                                              |
| OpenRouter Chat Model      | AI Language Model (OpenRouter)    | AI generation for title and structure          | Check inputs                    | Create post title and structure |                                              |
| Create post title and structure | Chain LLM                    | Generates post title and chapter structure     | OpenRouter Chat Model            | Structured Output Parser2, Create Drive folder |                                              |
| Structured Output Parser2  | Output Parser                    | Parses AI output into structured JSON          | Create post title and structure | Create Drive folder             |                                              |
| Create Drive folder        | Google Drive                    | Creates folder for saving content               | Structured Output Parser2        | Get output                     |                                              |
| Get output                 | Set                              | Prepares variables for chapter splitting       | Create Drive folder              | Split out chapters, Generate featured image |                                              |
| Split out chapters         | Split Out                       | Splits chapters for individual processing      | Get output                     | Merge, Loop Over Items          |                                              |
| Loop Over Items            | Split In Batches                | Processes chapters in batches                    | Split out chapters              | Merge chapters title and text, Wait1 |                                              |
| OpenRouter Chat Model1     | AI Language Model (OpenRouter)    | Generates chapter text                           | Loop Over Items                 | Write chapter text             |                                              |
| Write chapter text         | Chain LLM                      | Processes AI output into chapter text           | OpenRouter Chat Model1          | Loop Over Items1               |                                              |
| Structured Output Parser   | Output Parser                  | Parses chapter text output                        | Write chapter text              | Loop Over Items1               |                                              |
| Loop Over Items1           | Split In Batches               | Further batch processing for chapters            | Write chapter text              | Get title chapter and content, Wait |                                              |
| Get title chapter and content | Set                          | Prepares chapter title and content               | Loop Over Items1               | Merge chapters title and text  |                                              |
| Merge chapters title and text | Merge                       | Combines chapter titles and texts                | Get title chapter and content   | Combine into article           |                                              |
| Combine into article       | Code                           | Assembles full markdown article                   | Merge chapters title and text   | Final article in Markdown      |                                              |
| Final article in Markdown  | Set                            | Stores markdown article                           | Combine into article            | Markdown to HTML              |                                              |
| Markdown to HTML           | Markdown                       | Converts markdown to HTML                          | Final article in Markdown       | FInal article in HTML          |                                              |
| FInal article in HTML      | Set                            | Holds final HTML content                           | Markdown to HTML               | Create Doc                   |                                              |
| Create Doc                 | Google Docs                    | Creates Google Doc for backup                      | FInal article in HTML          | Save texts to Doc             |                                              |
| Save texts to Doc          | Google Docs                    | Saves article text to Google Doc                   | Create Doc                    | Post on Wordpress            |                                              |
| Post on Wordpress          | WordPress                      | Publishes blog post                                | Save texts to Doc             | Add featured image            |                                              |
| Add featured image         | Merge                         | Combines post and featured image data             | Post on Wordpress, Upload featured image | Upload featured image       |                                              |
| Generate featured image    | OpenAI (Langchain)             | Generates featured image                           | Get output                    | Upload featured image, Resize featured image |                                              |
| Resize featured image      | Edit Image                    | Resizes featured image                             | Generate featured image        | Merge2                       |                                              |
| Upload featured image      | HTTP Request                  | Uploads featured image to WordPress                | Resize featured image          | Update featured image meta data |                                              |
| Update featured image meta data | HTTP Request              | Updates metadata for featured image                | Upload featured image          | Set featured image for post   |                                              |
| Set featured image for post | HTTP Request                 | Assigns featured image to post                      | Update featured image meta data | Set excerpt                  |                                              |
| Set excerpt                | HTTP Request                  | Sets post excerpt                                   | Set featured image for post    | Edit Fields                  |                                              |
| Edit Fields               | Set                            | Finalizes post metadata                             | Set excerpt                   | None                        |                                              |
| Generate chapter image    | OpenAI (Langchain)             | Generates images for each chapter                    | Loop Over Items1              | Upload chapter images to Drive, Resize Image |                                              |
| Resize Image              | Edit Image                    | Resizes chapter images                               | Generate chapter image        | Upload chapter images        |                                              |
| Upload chapter images     | HTTP Request                  | Uploads chapter images to WordPress                   | Resize Image                 | Update image meta data       |                                              |
| Update image meta data    | HTTP Request                  | Updates metadata for chapter images                   | Upload chapter images        | Merge1                      |                                              |
| Upload chapter images to Drive | Google Drive              | Saves chapter images to Google Drive                  | Generate chapter image        | Merge1                      |                                              |
| Merge1                    | Merge                         | Merges chapter image upload and metadata flows       | Update image meta data, Upload chapter images to Drive | Merge chapters title and text |                                              |
| Get post sitemap          | HTTP Request                  | Downloads website sitemap XML                          | Check inputs                 | Get XML file                |                                              |
| Get XML file              | XML                           | Parses sitemap XML                                     | Get post sitemap             | Split out links             |                                              |
| Split out links           | Split Out                     | Splits sitemap URLs into individual links             | Get XML file                 | Limit1                     |                                              |
| Limit1                    | Limit                         | Limits internal links to 20 for SEO embedding          | Split out links              | Aggregate internal links    |                                              |
| Aggregate internal links  | Aggregate                     | Aggregates limited internal links into list             | Limit1                      | Merge                      |                                              |
| Merge                     | Merge                         | Merges internal links with chapter data                 | Aggregate internal links, Split out chapters | Loop Over Items            |                                              |
| Wait                      | Wait (disabled)               | Intended delay to avoid rate limits (disabled)          | Loop Over Items1             | Generate chapter image      |                                              |
| Wait1                     | Wait (disabled)               | Intended delay to avoid rate limits (disabled)          | Loop Over Items              | Generate chapter image      |                                              |
| Sticky Notes              | Sticky Note                   | Various notes for user guidance                         | N/A                         | N/A                        | See Section 5 for details                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Node: Google Sheets Trigger**  
   - Configure to watch a specific Google Sheet for new rows.  
   - Authenticate with Google Sheets credentials.

2. **Add Limit Node: Get the latest topic**  
   - Set limit to 1 to process only the newest row.

3. **Add Set Node: Settings**  
   - Map and normalize input fields (Topic, Style, Word Count, Chapters, etc.).

4. **Add If Node: Check inputs**  
   - Validate required fields (e.g., Topic not empty, Word Count > 0).  
   - Configure branches for success and failure.

5. **Add AI Language Model Node: OpenRouter Chat Model**  
   - Connect from Check inputs success branch.  
   - Configure OpenRouter credentials and model.  
   - Set prompt to generate blog post title and chapter structure based on inputs.

6. **Add Chain LLM Node: Create post title and structure**  
   - Use AI output to generate structured title and chapter list.  
   - Connect AI node output to this node.

7. **Add Structured Output Parser Node: Structured Output Parser2**  
   - Parse AI output into JSON structure.

8. **Add Google Drive Node: Create Drive folder**  
   - Create folder named after blog post title.  
   - Use parsed title as folder name.

9. **Add Set Node: Get output**  
   - Prepare variables for chapter processing and image generation.

10. **Add Split Out Node: Split out chapters**  
    - Split chapter list into individual chapters.

11. **Add Split In Batches Node: Loop Over Items**  
    - Batch process chapters to handle API limits.

12. **Add AI Language Model Node: OpenRouter Chat Model1**  
    - Generate chapter text per chapter.  
    - Use internal links and style parameters.

13. **Add Chain LLM Node: Write chapter text**  
    - Process AI output into clean chapter text.

14. **Add Structured Output Parser Node: Structured Output Parser**  
    - Parse chapter text output.

15. **Add Split In Batches Node: Loop Over Items1**  
    - Further batch processing for image generation.

16. **Add Set Node: Get title chapter and content**  
    - Prepare chapter title and content for merging.

17. **Add Merge Node: Merge chapters title and text**  
    - Combine all chapters into one structure.

18. **Add Code Node: Combine into article**  
    - Assemble full markdown article with embedded images and internal links.

19. **Add Set Node: Final article in Markdown**  
    - Store markdown article.

20. **Add Markdown Node: Markdown to HTML**  
    - Convert markdown to HTML.

21. **Add Set Node: FInal article in HTML**  
    - Store HTML content.

22. **Add Google Docs Node: Create Doc**  
    - Create Google Doc for article backup.

23. **Add Google Docs Node: Save texts to Doc**  
    - Save article content to Google Doc.

24. **Add WordPress Node: Post on Wordpress**  
    - Publish article with title, content, and category.

25. **Add Merge Node: Add featured image**  
    - Combine post data with featured image info.

26. **Add OpenAI Node: Generate featured image**  
    - Generate featured image from article title or summary.

27. **Add Edit Image Node: Resize featured image**  
    - Resize featured image to WordPress specs.

28. **Add Google Drive Node: Upload featured image to Drive**  
    - Save featured image to Google Drive folder.

29. **Add HTTP Request Node: Upload featured image**  
    - Upload featured image to WordPress media library.

30. **Add HTTP Request Node: Update featured image meta data**  
    - Update metadata for featured image.

31. **Add HTTP Request Node: Set featured image for post**  
    - Assign featured image to published post.

32. **Add HTTP Request Node: Set excerpt**  
    - Set post excerpt for SEO.

33. **Add Set Node: Edit Fields**  
    - Finalize post metadata.

34. **Add HTTP Request Node: Get post sitemap**  
    - Download website sitemap XML.

35. **Add XML Node: Get XML file**  
    - Parse sitemap XML.

36. **Add Split Out Node: Split out links**  
    - Extract individual internal links.

37. **Add Limit Node: Limit1**  
    - Limit internal links to 20.

38. **Add Aggregate Node: Aggregate internal links**  
    - Aggregate limited links for embedding.

39. **Add Merge Node: Merge**  
    - Merge internal links with chapter data.

40. **Add OpenAI Node: Generate chapter image**  
    - Generate images for each chapter.

41. **Add Edit Image Node: Resize Image**  
    - Resize chapter images.

42. **Add HTTP Request Node: Upload chapter images**  
    - Upload chapter images to WordPress.

43. **Add HTTP Request Node: Update image meta data**  
    - Update metadata for chapter images.

44. **Add Google Drive Node: Upload chapter images to Drive**  
    - Save chapter images to Google Drive.

45. **Add Merge Nodes: Merge1, Merge2**  
    - Manage merging of image upload and metadata flows.

46. **Add Wait Nodes (optional, disabled by default)**  
    - Insert delays to handle API rate limits if needed.

47. **Add Sticky Notes**  
    - Add user guidance notes as needed.

**Credentials Setup:**  
- Google Sheets: OAuth2 with access to the target spreadsheet.  
- Google Drive: OAuth2 with permissions to create folders and upload files.  
- WordPress: OAuth2 or Application Passwords for REST API access.  
- OpenRouter/OpenAI: API keys configured in Langchain nodes.

---

### 5. General Notes & Resources

| Note Content                                                                                             | Context or Link                                                                                   |
|--------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| The workflow is designed for full automation triggered by Google Sheets row addition.                   | Workflow Overview                                                                                |
| Supports flexible AI model selection via OpenRouter nodes for easy switching between GPT models.        | Unique Features                                                                                  |
| Includes rate limit handling with optional Wait nodes (disabled by default).                            | Unique Features                                                                                  |
| Limits internal links embedded to 20 by default to avoid SEO penalties.                                | Unique Features                                                                                  |
| Saves all generated content and images to Google Drive folder named after the blog post title.          | Unique Features                                                                                  |
| For best results, customize prompts and test with low-cost AI models before scaling.                   | Setup Steps                                                                                    |
| Workflow uses WordPress REST API for post publishing and media management; ensure API access is enabled. | Setup Steps                                                                                    |
| Video screenshots included in original documentation illustrate node layout and flow.                   | Provided screenshots (fileId:944, fileId:945)                                                   |
| Minh Duc TV provides expert consulting on n8n and AI automation workflows.                             | https://ducnguyen.cc/contact/                                                                   |

---

This document provides a comprehensive, structured reference for understanding, reproducing, and modifying the WordPress Auto-Blogging Pro workflow for SEO content automation.