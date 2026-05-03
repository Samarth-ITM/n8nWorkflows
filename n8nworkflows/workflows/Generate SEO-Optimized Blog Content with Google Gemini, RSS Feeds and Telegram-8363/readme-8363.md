Generate SEO-Optimized Blog Content with Google Gemini, RSS Feeds and Telegram

https://n8nworkflows.xyz/workflows/generate-seo-optimized-blog-content-with-google-gemini--rss-feeds-and-telegram-8363


# Generate SEO-Optimized Blog Content with Google Gemini, RSS Feeds and Telegram

### 1. Workflow Overview

This workflow automates the generation of SEO-optimized blog content by leveraging Google Gemini (PaLM) AI, multiple RSS feed sources, and Telegram for notifications. It is designed to:

- Continuously monitor various RSS feeds for new articles.
- Check if an article URL was already processed to avoid duplicates.
- Generate high-quality, SEO-focused blog posts in English using Google Gemini.
- Create AI-generated cover images based on blog content prompts.
- Upload images to Cloudflare R2 storage.
- Store blog data and processed URLs in AWS DynamoDB.
- Notify via Telegram in case of errors or duplicate articles.

The workflow is logically divided into these key blocks:

- **1.1 Input Reception and RSS Feed Monitoring:** Three RSS feed triggers poll feeds at different schedules, feeding into batch processing nodes.
- **1.2 Duplicate Detection and Processing Control:** For each RSS item, check DynamoDB if already processed, and control flow accordingly.
- **1.3 Content Generation with Google Gemini:** Generate blog content JSON using Google Gemini PaLM AI model.
- **1.4 Response Parsing and Content Preparation:** Parse and validate the AI-generated JSON blog content.
- **1.5 Image Generation and Upload:** Generate blog cover images with Gemini, rename the files, and upload to Cloudflare R2.
- **1.6 Final Assembly and Storage:** Consolidate all data, prepare final blog post object, and insert/update in DynamoDB.
- **1.7 Notifications via Telegram:** Send Telegram messages for duplicate detection, image generation errors, or content creation issues.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and RSS Feed Monitoring

- **Overview:** Continuously monitors three different RSS feeds with distinct poll frequencies and triggers batch processing of new items.
- **Nodes Involved:** RSS 1, RSS 2, RSS 3, Loop Over Items, Loop Over Items1, Loop Over Items2
- **Node Details:**

| Node Name | Type | Role | Configuration | Inputs | Outputs | Edge Cases / Failures |
|-----------|------|------|---------------|--------|---------|----------------------|
| RSS 1 | RSS Feed Read Trigger | Poll RSS feed every hour and every week | Empty feedUrl (to be set), poll times: every hour & every week | None (trigger node) | Loop Over Items2 | Feed URL invalid or unreachable, no new items |
| RSS 2 | RSS Feed Read Trigger | Poll RSS feed every week | Empty feedUrl, poll weekly | None (trigger node) | Loop Over Items | Same as RSS 1 |
| RSS 3 | RSS Feed Read Trigger | Poll RSS feed every week on Friday at noon | Empty feedUrl, poll weekly on Friday 12:00 | None (trigger node) | Loop Over Items1 | Same as RSS 1 |
| Loop Over Items | SplitInBatches | Process RSS feed items one by one | Default batch size (not specified) | Corresponding RSS node | Get an item or Merge3 | Empty batch, processing errors |
| Loop Over Items1 | SplitInBatches | Same as Loop Over Items | Default batch size | RSS 3 | Get an item or Merge3 | Same as above |
| Loop Over Items2 | SplitInBatches | Same as Loop Over Items | Default batch size | RSS 1 | Get an item or Merge3 | Same as above |

- **Notes:** Empty feed URLs indicate placeholders; must be configured with actual feed URLs.

---

#### 2.2 Duplicate Detection and Processing Control

- **Overview:** For each RSS item, check if the article URL already exists in DynamoDB to avoid duplicate blog creation.
- **Nodes Involved:** Get an item, Merge3, If
- **Node Details:**

| Node Name | Type | Role | Configuration | Inputs | Outputs | Edge Cases / Failures |
|-----------|------|------|---------------|--------|---------|----------------------|
| Get an item | AWS DynamoDB | Retrieves an item by article_url key from "processed_rss_links" table | Get operation, key: article_url from `$json.link` | Loop Over Items nodes | Merge3 | AWS auth failure, DynamoDB unavailability, missing keys |
| Merge3 | Merge | Combines outputs from Get an item and Loop Over Items nodes | Combine all mode, no additional options | Get an item, Loop Over Items nodes | If | Mismatched inputs, empty inputs |
| If | If | Checks if fetched Dynamo item is empty (article_url missing) | Condition: `$json.article_url` is empty string | Merge3 | Gemini Content Generation (if new), Telegram Debugger (if exists) | Expression errors, false positives |

- **Edge Cases:** Network issues accessing DynamoDB, malformed keys, false negative duplicate detection.

---

#### 2.3 Content Generation with Google Gemini

- **Overview:** Generates SEO-optimized blog content in JSON format using Google Gemini (PaLM) AI with a detailed prompt.
- **Nodes Involved:** Gemini Content Generation
- **Node Details:**

| Node Name | Type | Role | Configuration | Inputs | Outputs | Edge Cases / Failures |
|-----------|------|------|---------------|--------|---------|----------------------|
| Gemini Content Generation | Google Gemini (LangChain) | Generates blog post JSON content from RSS item link | Model: "models/gemini-2.5-flash"; Messages: detailed prompt specifying blog content structure including title, SEO metadata, body in markdown, image prompt; Language enforced to English | If node (new article) | Json Parser | API quota limits, network issues, malformed prompt, non-JSON output |

- **Notes:** Uses a complex prompt instructing Gemini to return a strict JSON format for blog content.

---

#### 2.4 Response Parsing and Content Preparation

- **Overview:** Parses raw Gemini AI responses into JSON blog objects, validates structure, and extracts relevant fields.
- **Nodes Involved:** Json Parser, Slug Selector
- **Node Details:**

| Node Name | Type | Role | Configuration | Inputs | Outputs | Edge Cases / Failures |
|-----------|------|------|---------------|--------|---------|----------------------|
| Json Parser | Code | Custom JS code to robustly parse Gemini response JSON | Multiple parsing strategies (direct parse, regex extraction, fixing common JSON errors); validates blog structure; extracts fields like title, slug, SEO data | Gemini Content Generation | Generate Image, Slug Selector, Merge1 | Parsing failures, unexpected response format, missing fields |
| Slug Selector | Set | Extracts and sets the slug string from parsed JSON | Assign slug = `$json.slug` | Json Parser | Merge | Missing slug in JSON |

- **Edge Cases:** Parsing errors due to malformed or incomplete AI output; missing mandatory blog fields.

---

#### 2.5 Image Generation and Upload

- **Overview:** Generates a blog cover image based on the AI’s image prompt, renames it using slug, and uploads it to Cloudflare R2 storage.
- **Nodes Involved:** Generate Image, Image Renamer, Upload Image to Cloudflare R2 Storage, If Statement
- **Node Details:**

| Node Name | Type | Role | Configuration | Inputs | Outputs | Edge Cases / Failures |
|-----------|------|------|---------------|--------|---------|----------------------|
| Generate Image | Google Gemini (LangChain) | Generates image from prompt string | Model: "models/gemini-2.5-flash-image-preview"; Prompt constructed dynamically with size and AI-provided image_generation_prompt | Json Parser | Image Renamer | API limits, bad prompts, image generation errors |
| Image Renamer | Set | Renames image file with slug and extension | fileName = `${slug}.${fileExtension}` | Generate Image | Upload Image to Cloudflare R2 Storage | Missing slug or fileExtension |
| Upload Image to Cloudflare R2 Storage | S3 | Uploads image file to bucket "saywize-media" | Upload operation, fileName from previous node | Image Renamer | If Statement | Upload failure, authorization errors, network issues |
| If Statement | If | Checks upload success flag | Condition: `$json.success == true` | Upload Image node | Merge1 (success), Telegram Debugger1 (failure) | False negatives, missing success flag |

- **Notes:** Uses Cloudflare R2 as S3-compatible storage. Failure in image upload triggers Telegram notification.

---

#### 2.6 Final Assembly and Storage

- **Overview:** Combines all blog content and image info, finalizes the blog post data, converts to JSON file, and stores blog metadata in DynamoDB.
- **Nodes Involved:** Merge, Last Content Detection, Convert to File, Create or update an item
- **Node Details:**

| Node Name | Type | Role | Configuration | Inputs | Outputs | Edge Cases / Failures |
|-----------|------|------|---------------|--------|---------|----------------------|
| Merge | Merge | Combines blog content and image upload results | Combine all mode | Slug Selector, If Statement (upload success) | Last Content Detection, Generate Image | Input mismatch |
| Last Content Detection | Set | Assigns final blog post fields including title, description, slug, body, author, category, SEO metadata, image URL, article URL | Uses expressions to map JSON fields from merged input | Merge | Convert to File | Missing fields, expression errors |
| Convert to File | ConvertToFile | Converts blog JSON to file with filename `${slug}.json` | Operation: toJson | Last Content Detection | Edit Chat ID | File conversion errors |
| Create or update an item | AWS DynamoDB | Inserts or updates blog post metadata in "processed_rss_links" table | Operation: create/update, fields include article URL | Gemini Content Generation | None | DynamoDB auth issues, data validation errors |

- **Notes:** The created blog post is marked as draft with a timestamp and author. The article_url field is used as a key.

---

#### 2.7 Notifications via Telegram

- **Overview:** Sends Telegram messages for duplicate detection, image generation errors, or blog creation issues to notify the user.
- **Nodes Involved:** Telegram Debugger, Telegram Debugger1, Telegram Debugger2, Edit Chat ID
- **Node Details:**

| Node Name | Type | Role | Configuration | Inputs | Outputs | Edge Cases / Failures |
|-----------|------|------|---------------|--------|---------|----------------------|
| Telegram Debugger | Telegram | Sends message if article already exists (duplicate) | Text: "Since the article already exists, the blog post could not be created: {{ $json.article_url }}" | If (duplicate) | None | Telegram API limits, invalid chat ID |
| Telegram Debugger1 | Telegram | Sends message if image creation/upload fails | Text: "An error occurred during image creation." | If Statement (upload failure) | None | Same as above |
| Telegram Debugger2 | Telegram | Sends message if blog creation fails | Text: "I encountered a problem while creating a blog." | If Statement2 (error branch) | None | Same as above |
| Edit Chat ID | Telegram | Sends the final blog JSON file as a document to Telegram chat | Operation: sendDocument, binaryData enabled | Convert to File | None | Invalid chat ID, file size limits |

- **Notes:** Telegram credentials must be configured properly. Sticky note included in the workflow explains bot creation and user ID retrieval.

---

### 3. Summary Table

| Node Name                     | Node Type                           | Functional Role                      | Input Node(s)                 | Output Node(s)                   | Sticky Note                                                                            |
|-------------------------------|-----------------------------------|------------------------------------|------------------------------|---------------------------------|----------------------------------------------------------------------------------------|
| RSS 1                         | RSS Feed Read Trigger              | Poll RSS feed hourly & weekly      | None                         | Loop Over Items2                 |                                                                                        |
| RSS 2                         | RSS Feed Read Trigger              | Poll RSS feed weekly               | None                         | Loop Over Items                  |                                                                                        |
| RSS 3                         | RSS Feed Read Trigger              | Poll RSS feed weekly on Friday     | None                         | Loop Over Items1                 |                                                                                        |
| Loop Over Items                | SplitInBatches                    | Process RSS items batchwise        | RSS 2                        | Get an item, Merge3              |                                                                                        |
| Loop Over Items1               | SplitInBatches                    | Process RSS items batchwise        | RSS 3                        | Get an item, Merge3              |                                                                                        |
| Loop Over Items2               | SplitInBatches                    | Process RSS items batchwise        | RSS 1                        | Get an item, Merge3              |                                                                                        |
| Get an item                   | AWS DynamoDB                      | Check if article URL processed     | Loop Over Items nodes         | Merge3                         |                                                                                        |
| Merge3                        | Merge                            | Combine DynamoDB check & RSS item  | Get an item, Loop Over Items  | If                            |                                                                                        |
| If                           | If                               | Check if article is new             | Merge3                       | Gemini Content Generation, Telegram Debugger |                                                                                        |
| Gemini Content Generation      | Google Gemini (LangChain)         | Generate SEO blog content JSON     | If                          | Json Parser                    |                                                                                        |
| Json Parser                   | Code                             | Parse & validate Gemini JSON output| Gemini Content Generation    | Generate Image, Slug Selector, Merge1 |                                                                                        |
| Slug Selector                | Set                              | Extract slug from parsed JSON      | Json Parser                  | Merge                          |                                                                                        |
| Generate Image                | Google Gemini (LangChain)         | Generate blog cover image          | Json Parser                  | Image Renamer                  |                                                                                        |
| Image Renamer                | Set                              | Rename image file based on slug    | Generate Image               | Upload Image to Cloudflare R2 Storage |                                                                                        |
| Upload Image to Cloudflare R2 Storage | S3                         | Upload image to Cloudflare R2      | Image Renamer                | If Statement                   |                                                                                        |
| If Statement                 | If                               | Check image upload success         | Upload Image to Cloudflare R2 Storage | Merge1, Telegram Debugger1     |                                                                                        |
| Merge                        | Merge                            | Combine image and slug data        | Slug Selector, If Statement  | Last Content Detection, Generate Image |                                                                                        |
| Last Content Detection       | Set                              | Finalize blog content fields       | Merge                        | Convert to File                |                                                                                        |
| Convert to File              | ConvertToFile                    | Convert blog content to JSON file  | Last Content Detection       | Edit Chat ID                  |                                                                                        |
| Create or update an item      | AWS DynamoDB                      | Store blog metadata                | Gemini Content Generation    | None                         |                                                                                        |
| Telegram Debugger            | Telegram                         | Notify duplicate article           | If                          | None                         |                                                                                        |
| Telegram Debugger1           | Telegram                         | Notify image creation/upload error | If Statement                | None                         |                                                                                        |
| Telegram Debugger2           | Telegram                         | Notify blog creation error         | If Statement2               | None                         |                                                                                        |
| Edit Chat ID                 | Telegram                         | Send blog JSON file to Telegram    | Convert to File             | None                         |                                                                                        |
| Merge1                       | Merge                            | Combine image upload and slug paths| Json Parser, If Statement   | Merge                        |                                                                                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create RSS Feed Triggers:**
   - Add three `RSS Feed Read Trigger` nodes named RSS 1, RSS 2, RSS 3.
   - Configure RSS 1 to poll every hour and every week.
   - Configure RSS 2 to poll every week.
   - Configure RSS 3 to poll every Friday at 12:00.
   - Set actual RSS feed URLs in each node's `feedUrl` parameter.

2. **Add Batch Processing Nodes:**
   - Add three `SplitInBatches` nodes named Loop Over Items, Loop Over Items1, Loop Over Items2.
   - Connect RSS 1 → Loop Over Items2; RSS 2 → Loop Over Items; RSS 3 → Loop Over Items1.

3. **Add DynamoDB Get Node:**
   - Add `AWS DynamoDB` node named Get an item.
   - Configure with AWS credentials.
   - Set operation to "get".
   - Set key `article_url` to `{{$json.link}}`.
   - Connect Loop Over Items*, Loop Over Items1, Loop Over Items2 (batch outputs) to Get an item.

4. **Add Merge Node:**
   - Add `Merge` node named Merge3.
   - Set mode to "combine all".
   - Connect Get an item and batch processing nodes to Merge3.

5. **Add If Node for Duplicate Check:**
   - Add `If` node named If.
   - Condition: Check if `{{$json.article_url}}` is empty string.
   - Connect Merge3 output to If node.

6. **Add Google Gemini Content Generation Node:**
   - Add `Google Gemini (LangChain)` node named Gemini Content Generation.
   - Set model ID to "models/gemini-2.5-flash".
   - Use the provided detailed system and user messages prompt that instructs Gemini to generate SEO blog JSON.
   - Connect If node (true branch) to this node.
   - Set Google Palm API credentials.

7. **Add Custom Code Node for Parsing:**
   - Add a `Code` node named Json Parser.
   - Paste the provided JavaScript code that robustly parses Gemini response JSON and validates it.
   - Connect Gemini Content Generation output to Json Parser.

8. **Add Slug Selector Node:**
   - Add `Set` node named Slug Selector.
   - Assign slug field: `slug = {{$json.slug}}`.
   - Connect Json Parser to Slug Selector.

9. **Add Google Gemini Image Generation Node:**
   - Add another `Google Gemini (LangChain)` node named Generate Image.
   - Model: "models/gemini-2.5-flash-image-preview" (image generation).
   - Prompt: `**Image Size: 1200x630 aspect Ratio 1.91:1**\n{{$json.image_generation_prompt}}`.
   - Connect Json Parser output to Generate Image.
   - Use same Google Palm API credentials.

10. **Add Image Renamer Node:**
    - Add `Set` node named Image Renamer.
    - Assign `fileName = {{$json.slug}}.{{$json.fileExtension}}`.
    - Include other fields.
    - Connect Generate Image to Image Renamer.

11. **Add Cloudflare R2 Upload Node:**
    - Add `S3` node named Upload Image to Cloudflare R2 Storage.
    - Configure with Cloudflare R2 credentials.
    - Bucket: "saywize-media".
    - File name: `{{$json.fileName}}`.
    - Operation: upload.
    - Connect Image Renamer to this node.

12. **Add If Node to Check Upload Success:**
    - Add `If` node named If Statement.
    - Condition: `$json.success` is true.
    - Connect Upload Image node to this If node.

13. **Add Merge Node:**
    - Add `Merge` node named Merge1.
    - Mode: combine all.
    - Connect Slug Selector and If Statement (true branch) to Merge1.

14. **Add Merge Node:**
    - Add `Merge` node named Merge.
    - Mode: combine all.
    - Connect Merge1 and If Statement (true branch) to Merge.

15. **Add Set Node for Final Content:**
    - Add `Set` node named Last Content Detection.
    - Assign blog post fields: title, description, slug, body, author (e.g., "Baris's Agent"), category, SEO metaTitle, metaDescription, image_url (constructed from slug), article_url (from Merge3).
    - Connect Merge to this node.

16. **Add Convert to File Node:**
    - Add `ConvertToFile` node named Convert to File.
    - Operation: toJson.
    - FileName: `{{$json.slug}}.json`.
    - Connect Last Content Detection to this node.

17. **Add Telegram Node to Send File:**
    - Add `Telegram` node named Edit Chat ID.
    - Operation: sendDocument.
    - Enable binary data.
    - Configure Telegram credentials.
    - Connect Convert to File to this node.

18. **Add DynamoDB Create or Update Node:**
    - Add `AWS DynamoDB` node named Create or update an item.
    - Configure with AWS credentials.
    - Operation: create/update.
    - Table: "processed_rss_links".
    - Fields: article_url from `$json.link` or blog URL.
    - Connect Gemini Content Generation output (true branch) to this node.

19. **Add Telegram Debug Nodes for Errors and Duplicates:**
    - Add Telegram nodes named Telegram Debugger (duplicate), Telegram Debugger1 (image error), Telegram Debugger2 (blog creation error).
    - Configure messages as specified.
    - Connect appropriate If node false branches to these for notifications.

20. **Set Credentials:**
    - Configure Google Palm API credentials for Gemini nodes.
    - Configure AWS credentials for DynamoDB nodes.
    - Configure Cloudflare R2 credentials for upload node.
    - Configure Telegram API credentials for Telegram nodes.

21. **Add Sticky Note:**
    - Add a sticky note with details on Telegram bot creation and obtaining chat IDs, linking https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.telegram/message-operations/#send-sticker.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
|--------------|-----------------|
| Telegram bot creation instructions, including use of BotFather and Userinfobot for obtaining User IDs. | https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.telegram/message-operations/#send-sticker |
| The workflow uses Google Gemini models both for text content and image generation via LangChain integration. | n8n Google Gemini node documentation |
| Cloudflare R2 used as S3-compatible object storage for images, requires proper credential setup. | Cloudflare R2 docs |
| DynamoDB table "processed_rss_links" is used both to prevent duplicate processing and to store blog metadata. | AWS DynamoDB documentation |
| The custom code node robustly handles parsing of AI-generated JSON responses, including error handling and validation to ensure well-formed blog data. | Inline JavaScript code in Json Parser node |

---

**Disclaimer:** The provided text is extracted exclusively from an automated workflow created with n8n, a tool for integration and automation. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.