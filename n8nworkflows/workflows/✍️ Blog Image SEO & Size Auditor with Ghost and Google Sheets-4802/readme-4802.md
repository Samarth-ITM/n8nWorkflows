✍️ Blog Image SEO & Size Auditor with Ghost and Google Sheets

https://n8nworkflows.xyz/workflows/---blog-image-seo---size-auditor-with-ghost-and-google-sheets-4802


# ✍️ Blog Image SEO & Size Auditor with Ghost and Google Sheets

### 1. Workflow Overview

This workflow is designed to audit blog images from a Ghost CMS blog, focusing on SEO attributes and image file sizes. Its main use case is to extract blog posts, collect all images within these posts along with their metadata (such as alt text), and then verify image file sizes and SEO-friendly filename conventions. The results are systematically logged into a Google Sheets document for easy review and further analysis.

The workflow comprises four logical blocks:

- **1.1 Trigger and Blog Post Extraction:** Initiates the workflow manually and fetches blog posts from Ghost CMS.
- **1.2 Image Collection from Blog Posts:** Parses the blog post content to extract all images and their alt texts.
- **1.3 Image Metadata Retrieval and Size Verification:** Downloads each image to gather file size and format info; checks if the image size is within acceptable limits and if filenames are SEO-friendly.
- **1.4 Data Logging in Google Sheets:** Saves extracted image information and audit results into Google Sheets for tracking and reporting.

---

### 2. Block-by-Block Analysis

#### Block 1.1: Trigger and Blog Post Extraction

- **Overview:**  
  This block starts the workflow with a manual trigger and extracts blog posts from Ghost CMS, limiting to one post by default.

- **Nodes Involved:**  
  - When clicking ‘Test workflow’ (Manual Trigger)  
  - Extract Blog Posts (Ghost node)  
  - Extract Post Content (Set node)

- **Node Details:**  

  - **When clicking ‘Test workflow’**  
    - *Type:* Manual Trigger  
    - *Role:* Workflow entry point initiated by user action.  
    - *Configuration:* Default manual trigger with no parameters.  
    - *Connections:* Outputs to "Extract Blog Posts".  
    - *Failure Types:* None typical; user must trigger manually.  
    - *Notes:* Simple trigger, no setup needed.

  - **Extract Blog Posts**  
    - *Type:* Ghost CMS Integration Node  
    - *Role:* Retrieves blog posts from a Ghost blog via Content API.  
    - *Configuration:*  
      - Operation: getAll  
      - Limit: 1 (fetches one post)  
      - Credentials: Ghost Content API credentials required.  
    - *Connections:* Receives from manual trigger; outputs to "Extract Post Content".  
    - *Failure Types:* API authentication errors, network timeouts, invalid credentials.  
    - *Notes:* Requires Ghost API key and blog URL.

  - **Extract Post Content**  
    - *Type:* Set Node  
    - *Role:* Extracts and sets useful post properties from raw Ghost data for downstream processing.  
    - *Configuration:* Assigns fields with expressions:
      - id = {{$json.id}}  
      - title = {{$json.title}}  
      - featured_image = {{$json.feature_image}}  
      - excerpt = {{$json.excerpt}}  
      - content = {{$json.html}} (full HTML content)  
      - link = {{$json.url}}  
    - *Connections:* Input from "Extract Blog Posts", output to "Loop Over Items".  
    - *Failure Types:* Expression failures if data missing or field names change.

- **Sticky Note:**  
  - Describes the Ghost API setup and how to configure extraction parameters.  
  - Link: [Ghost Node Documentation](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.ghost)

---

#### Block 1.2: Image Collection from Blog Posts

- **Overview:**  
  This block iterates over blog posts to extract all images (URLs and alt texts) from the HTML content and appends this data as records in a Google Sheet.

- **Nodes Involved:**  
  - Loop Over Items (SplitInBatches)  
  - Collect All Images (Code)  
  - Add Records (Google Sheets)

- **Node Details:**

  - **Loop Over Items**  
    - *Type:* SplitInBatches  
    - *Role:* Processes blog posts one batch at a time to handle large datasets efficiently.  
    - *Configuration:* Default batch size (implied).  
    - *Connections:* Input from "Extract Post Content"; outputs to "Collect All Images" and "Collect Records by Id".  
    - *Failure Types:* Potential batch processing delays.

  - **Collect All Images**  
    - *Type:* Code (JavaScript)  
    - *Role:* Parses the HTML content of each post to extract every `<img>` tag’s `src` and `alt` attributes.  
    - *Configuration:* Regex used to find images: `<img[^>]*src="([^"]+)"[^>]*alt="([^"]*)"?[^>]*>`  
    - *Outputs:* Array of JSON objects with: articleId, articleTitle, article_url, image_url, alt_text.  
    - *Connections:* Input from "Loop Over Items"; output to "Add Records".  
    - *Failure Types:* Edge cases include missing alt attributes, complex HTML structures, or non-standard image tags.

  - **Add Records**  
    - *Type:* Google Sheets  
    - *Role:* Appends extracted image data to a specified Google Sheet.  
    - *Configuration:*  
      - Operation: append  
      - Document ID: Google Sheet ID provided  
      - Sheet Name: Numeric sheet identifier  
      - Column mapping: articleId, articleTitle, article_url, image_url, alt_text  
      - Credentials: Google Sheets OAuth2 credentials required  
    - *Connections:* Input from "Collect All Images".  
    - *Failure Types:* Auth errors, sheet access issues, quota limits.  
    - *Notes:* Ensures raw image data is logged for further processing.

- **Sticky Note:**  
  - Explains how to configure Google Sheets node and map fields.  
  - Link: [Google Sheets Node Documentation](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.googlesheets)

---

#### Block 1.3: Image Metadata Retrieval and Size Verification

- **Overview:**  
  This block retrieves previously recorded images from the Google Sheet by article ID, downloads each image, extracts file size and format, verifies SEO-friendly filename conventions, and flags if the size exceeds 500 KB.

- **Nodes Involved:**  
  - Collect Records by Id (Google Sheets)  
  - Loop Over Items1 (SplitInBatches)  
  - Get Image (HTTP Request)  
  - Extract Image Info (Code)  
  - Results Images (Google Sheets)

- **Node Details:**

  - **Collect Records by Id**  
    - *Type:* Google Sheets  
    - *Role:* Filters and retrieves image records matching the current article ID.  
    - *Configuration:*  
      - Filter on column "articleId" equals {{$json.id}}  
      - Document and sheet same as previous Google Sheets node  
      - Credentials: Google Sheets OAuth2 credentials  
    - *Connections:* Input from "Loop Over Items" (second output); outputs to "Loop Over Items1".  
    - *Failure Types:* Filter errors, auth issues.

  - **Loop Over Items1**  
    - *Type:* SplitInBatches  
    - *Role:* Batches image records for sequential processing.  
    - *Configuration:* Default batch size.  
    - *Connections:* Input from "Collect Records by Id"; outputs to "Get Image".  
    - *Failure Types:* Processing delays.

  - **Get Image**  
    - *Type:* HTTP Request  
    - *Role:* Downloads each image file as a binary to analyze.  
    - *Configuration:*  
      - URL: {{$json.image_url}}  
      - Response format: file (binary)  
    - *Connections:* Input from "Loop Over Items1"; output to "Extract Image Info".  
    - *Failure Types:* HTTP errors (404, timeouts), large files causing delays.

  - **Extract Image Info**  
    - *Type:* Code (JavaScript)  
    - *Role:* Extracts image metadata such as file name, size in KB, file format, and SEO filename check; flags oversized images.  
    - *Configuration:*  
      - Runs once per item  
      - Checks file size ≤ 500 KB to set `size_ok` flag  
      - Regex to verify filename SEO compliance (lowercase letters, numbers, dashes, minimum length 3)  
    - *Outputs:* JSON with file_name, file_size_kb, format, filename_seo (bool), size_ok (bool)  
    - *Connections:* Input from "Get Image"; output to "Results Images".  
    - *Failure Types:* Missing binary data, unexpected file names.

  - **Results Images**  
    - *Type:* Google Sheets  
    - *Role:* Appends or updates image metadata records in Google Sheets, matching on image URL.  
    - *Configuration:*  
      - Operation: appendOrUpdate  
      - Matching column: image_url  
      - Fields: format, size_ok, image_url, file_size_kb, filename_seo  
      - Credentials: Google Sheets OAuth2  
    - *Connections:* Input from "Extract Image Info".  
    - *Failure Types:* Auth errors, update conflicts.

- **Sticky Note:**  
  - Explains the image download and metadata extraction process, including Google Sheets update setup.  
  - Link: [Google Sheets Node Documentation](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.googlesheets)

---

### 3. Summary Table

| Node Name                 | Node Type           | Functional Role                               | Input Node(s)                  | Output Node(s)                          | Sticky Note                                                                                                                                                |
|---------------------------|---------------------|-----------------------------------------------|-------------------------------|---------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger      | Workflow start trigger                         | -                             | Extract Blog Posts                    | This workflow uses simple trigger. Nothing to do.                                                                                                         |
| Extract Blog Posts         | Ghost               | Fetch blog posts from Ghost CMS                | When clicking ‘Test workflow’ | Extract Post Content                  | Describes Ghost API setup and extraction parameters. [Learn more](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.ghost)                 |
| Extract Post Content       | Set                 | Extracts post metadata and content             | Extract Blog Posts             | Loop Over Items                      | Same as above                                                                                                                                              |
| Loop Over Items            | SplitInBatches      | Batch processing of blog posts                  | Extract Post Content           | Collect All Images, Collect Records by Id | Describes first loop collecting images into Google Sheets. [Learn more](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.googlesheets)    |
| Collect All Images         | Code                | Extract images and alt text from HTML          | Loop Over Items                | Add Records                         | Same as above                                                                                                                                              |
| Add Records                | Google Sheets       | Append image data to Google Sheet               | Collect All Images             | Loop Over Items (2nd output)          | Same as above                                                                                                                                              |
| Collect Records by Id      | Google Sheets       | Retrieve images by article ID from Google Sheet | Loop Over Items (2nd output)  | Loop Over Items1                    |                                                                                                                                                            |
| Loop Over Items1           | SplitInBatches      | Batch processing of image records               | Collect Records by Id          | Get Image                          | Describes second loop for downloading images and extracting info. [Learn more](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.googlesheets)|
| Get Image                 | HTTP Request         | Download image binary data                       | Loop Over Items1               | Extract Image Info                  | Same as above                                                                                                                                              |
| Extract Image Info         | Code                | Extract file size, format, SEO filename check  | Get Image                     | Results Images                     | Same as above                                                                                                                                              |
| Results Images             | Google Sheets       | Append or update image metadata in Google Sheet | Extract Image Info             | Loop Over Items1 (2nd output)          | Same as above                                                                                                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a Manual Trigger Node**  
   - Name: `When clicking ‘Test workflow’`  
   - No parameters needed. This node will manually start the workflow.

3. **Add a Ghost Node**  
   - Name: `Extract Blog Posts`  
   - Operation: `getAll`  
   - Limit: `1` (or set as needed)  
   - Credentials: Add and select your Ghost Content API credentials (requires API key from your Ghost blog).  
   - Connect output of Manual Trigger to this node.

4. **Add a Set Node**  
   - Name: `Extract Post Content`  
   - Purpose: Extract and simplify post data fields.  
   - Define fields:  
     - id = `{{$json.id}}`  
     - title = `{{$json.title}}`  
     - featured_image = `{{$json.feature_image}}`  
     - excerpt = `{{$json.excerpt}}`  
     - content = `{{$json.html}}`  
     - link = `{{$json.url}}`  
   - Connect from Ghost node.

5. **Add SplitInBatches Node**  
   - Name: `Loop Over Items`  
   - Default batch size is acceptable.  
   - Connect from Set node.

6. **Add a Code Node**  
   - Name: `Collect All Images`  
   - Paste JavaScript code to extract images and alt texts using regex from `content` field.  
   - Input: items from `Loop Over Items`.  
   - Output: array of JSON objects with article and image info.  
   - Connect from `Loop Over Items` main output.

7. **Add Google Sheets Node**  
   - Name: `Add Records`  
   - Operation: `append`  
   - Document ID: input your Google Sheets document ID.  
   - Sheet Name: enter the target sheet (by name or ID).  
   - Columns mapping: articleId, articleTitle, article_url, image_url, alt_text mapped from incoming JSON.  
   - Credentials: Google Sheets OAuth2 credentials configured.  
   - Connect from `Collect All Images`.

8. **Connect the second output of `Loop Over Items` to a Google Sheets Node**  
   - Name: `Collect Records by Id`  
   - Operation: `read` with filter on column `articleId` equals `{{$json.id}}`  
   - Document and Sheet same as above.  
   - Credentials: Google Sheets OAuth2 credentials.  
   - Connect output to `Loop Over Items1`.

9. **Add SplitInBatches Node**  
   - Name: `Loop Over Items1`  
   - Default batch size.  
   - Connect from `Collect Records by Id`.

10. **Add HTTP Request Node**  
    - Name: `Get Image`  
    - Method: GET  
    - URL: `{{$json.image_url}}`  
    - Response Format: file (to download the image binary).  
    - Connect from `Loop Over Items1`.

11. **Add Code Node**  
    - Name: `Extract Image Info`  
    - Run once per item.  
    - JavaScript to extract file name, file size in KB, file format, check if filename matches SEO pattern `/[a-z0-9-]{3,}/`, and flag size_ok if ≤ 500KB.  
    - Connect from `Get Image`.

12. **Add Google Sheets Node**  
    - Name: `Results Images`  
    - Operation: `appendOrUpdate`  
    - Matching Column: `image_url`  
    - Fields: format, size_ok, image_url, file_size_kb, filename_seo mapped from code output.  
    - Document ID and Sheet same as previous.  
    - Credentials: Google Sheets OAuth2.  
    - Connect from `Extract Image Info`.

13. **Validate and test the workflow flow by manually triggering.**

---

### 5. General Notes & Resources

| Note Content                                                                                                          | Context or Link                                                                                               |
|-----------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------|
| Workflow uses Ghost Content API; ensure API key has read access to posts.                                             | [Ghost Node Documentation](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.ghost)          |
| Google Sheets integration requires OAuth2 credentials with edit permissions on target spreadsheet.                    | [Google Sheets Node Documentation](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.googlesheets) |
| Images larger than 500 KB are flagged as size not OK to help optimize page load performance.                          | Internal workflow logic                                                                                       |
| SEO filename check uses regex to enforce lowercase alphanumeric and dash patterns with minimum length 3 for filenames. | Internal workflow logic                                                                                       |

---

**Disclaimer:**  
The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and public.