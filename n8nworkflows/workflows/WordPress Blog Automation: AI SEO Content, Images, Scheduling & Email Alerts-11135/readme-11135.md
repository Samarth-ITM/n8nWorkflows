WordPress Blog Automation: AI SEO Content, Images, Scheduling & Email Alerts

https://n8nworkflows.xyz/workflows/wordpress-blog-automation--ai-seo-content--images--scheduling---email-alerts-11135


# WordPress Blog Automation: AI SEO Content, Images, Scheduling & Email Alerts

### 1. Workflow Overview

This workflow automates the generation, scheduling, and publication of SEO-optimized blog posts on a WordPress site using data sourced from a Google Spreadsheet. It runs on a daily schedule, checks for posts scheduled for the current day, generates rich content using OpenAI, fetches and processes internal links from the sitemap, creates and uploads SEO-friendly feature images, publishes the posts as drafts on WordPress, and finally sends notification emails to the administrator.

**Target Use Cases:**  
- Automated content marketing for WordPress blogs  
- SEO content generation with AI assistance  
- Scheduled blog post publishing with internal linking  
- Automated feature image creation and upload  
- Administrative notification upon draft creation  

**Logical Blocks:**

- **1.1 Trigger and Input Reception:** Schedule trigger and spreadsheet data retrieval with date filtering  
- **1.2 Sitemap Processing:** Download, parse sitemap XML, extract URLs for internal links  
- **1.3 AI Content Generation:** Generate structured blog post content with OpenAI and convert to HTML  
- **1.4 Feature Image Preparation:** Retrieve base image, format title text, and create SEO-friendly feature image  
- **1.5 WordPress Post Creation:** Upload feature image, create draft post, add metadata and excerpt  
- **1.6 SEO Metadata and Image Meta:** Add Yoast SEO metadata and feature image meta information  
- **1.7 Post Rendering and Notification:** Build final HTML post template and send notification email  

---

### 2. Block-by-Block Analysis

---

#### 1.1 Trigger and Input Reception

**Overview:**  
This block triggers the workflow daily at 4 AM, retrieves blog post scheduling and SEO data from a Google Sheet, and filters the data to only process posts scheduled for the current date.

**Nodes Involved:**  
- Schedule Trigger1  
- Get post information from Spreadsheet  
- Compare date of spreadsheet and todays  
- Edit Fields  

**Node Details:**

- **Schedule Trigger1**  
  - Type: Schedule Trigger  
  - Role: Initiates workflow daily at 4 AM  
  - Configuration: Trigger at hour 4 (4:00 AM daily)  
  - Inputs: None (start node)  
  - Outputs: Next node: Get post information from Spreadsheet  
  - Edge cases: Missed trigger if n8n instance down or paused  

- **Get post information from Spreadsheet**  
  - Type: Google Sheets  
  - Role: Retrieves post data (keywords, publish date, category, links, etc.) from a specific Google Sheet  
  - Configuration: Reads sheet "Sheet1" from document ID `1GXRVPjsGPK5prQVvZUjg1fHKInvjqbEdAW09uCcq8sY`  
  - Inputs: Schedule trigger output  
  - Outputs: Next node: Compare date of spreadsheet and todays  
  - Edge cases: Auth errors, Google API rate limits, missing sheet or document  

- **Compare date of spreadsheet and todays**  
  - Type: If  
  - Role: Filters rows to only those where "Date to be published" matches current date (ISO date comparison)  
  - Configuration: Checks if spreadsheet date contains today's date in ISO (YYYY-MM-DD) format based on trigger timestamp  
  - Inputs: Spreadsheet rows  
  - Outputs: True branch to Edit Fields, False branch ends flow for unmatched rows  
  - Edge cases: Date format inconsistencies, timezone mismatches  

- **Edit Fields**  
  - Type: Set  
  - Role: Prepares and normalizes all necessary data fields for downstream processing, including keywords, links, category, author, and email notification address  
  - Configuration: Assigns multiple custom fields by extracting from spreadsheet JSON and previous nodes (e.g., Primary keyword, Secondary keyword, Word Count, Style, Customer Offer, URLs, etc.)  
  - Inputs: Filtered spreadsheet row  
  - Outputs: Next node: GET information from sitemap  
  - Edge cases: Missing or malformed data fields in spreadsheet  

---

#### 1.2 Sitemap Processing

**Overview:**  
Fetches the WordPress sitemap XML, parses it, and extracts all blog post URLs to use for internal linking in generated articles.

**Nodes Involved:**  
- GET information from sitemap  
- XML  
- Split Out  
- Aggregate  

**Node Details:**

- **GET information from sitemap**  
  - Type: HTTP Request  
  - Role: Downloads the sitemap XML from the WordPress site (URL constructed dynamically from Website field)  
  - Configuration: GET request to `${Website}/post-sitemap.xml`  
  - Inputs: Output from Edit Fields  
  - Outputs: Next node: XML  
  - Edge cases: HTTP errors (404 if sitemap missing), slow response, invalid XML  

- **XML**  
  - Type: XML Parser  
  - Role: Parses the downloaded XML string into structured JSON for ease of processing  
  - Configuration: Parses data from `data` property of HTTP response  
  - Inputs: HTTP response  
  - Outputs: Next node: Split Out  
  - Edge cases: Malformed XML, large sitemap causing performance issues  

- **Split Out**  
  - Type: Split Out  
  - Role: Splits the `urlset.url` array of URLs into individual items for aggregation  
  - Configuration: Splits on field `urlset.url`  
  - Inputs: Parsed XML JSON  
  - Outputs: Next node: Aggregate  
  - Edge cases: Empty sitemap, unexpected XML structure  

- **Aggregate**  
  - Type: Aggregate  
  - Role: Aggregates the URLs back into a single array named "Blog URL List" for internal linking use  
  - Configuration: Aggregates field `loc` renaming output to "Blog URL List"  
  - Inputs: Individual URL items  
  - Outputs: Next node: Generate all the necessary information of post  
  - Edge cases: No URLs found, aggregation errors  

---

#### 1.3 AI Content Generation

**Overview:**  
Generates a fully structured SEO-friendly blog post using OpenAI GPT-4o-mini, including title, subtitle, chapters, introduction, conclusion, FAQ, metadata, and call to action. Converts the AI JSON output into final HTML content.

**Nodes Involved:**  
- Generate all the necessary information of post  
- Code in JavaScript  
- Code in JavaScript1  

**Node Details:**

- **Generate all the necessary information of post**  
  - Type: OpenAI (Langchain node)  
  - Role: Sends a detailed prompt to OpenAI GPT-4o-mini to produce JSON with all blog post components based on input keywords and parameters  
  - Configuration:  
    - Model: GPT-4o-mini  
    - Prompt includes constraints on style, chapter count, internal and external linking, SEO metadata, FAQ, call to action integration, and HTML formatting (limited tags)  
    - Inputs: Aggregated blog URLs, fields from Edit Fields node (keywords, style, offers, links, etc.)  
  - Outputs: JSON structured blog content  
  - Edge cases: API quota limits, prompt errors, incomplete or malformed JSON, rate limits, timeout  

- **Code in JavaScript**  
  - Type: Code (JavaScript)  
  - Role: Converts the OpenAI JSON response into a single HTML article string for WordPress content  
  - Configuration:  
    - Extracts introduction, chapters (with titles and content), conclusions, and FAQ section  
    - Formats chapters with `<h2>` and FAQ with `<h3>` and `<p>` tags  
    - Handles cases where FAQ is an array, object, or plain text  
  - Inputs: OpenAI output JSON  
  - Outputs: JSON with field `article` containing full HTML content  
  - Edge cases: Missing fields in JSON, malformed content, empty FAQ sections  

- **Code in JavaScript1**  
  - Type: Code (JavaScript)  
  - Role: Wraps the blog post title text to produce line breaks for better appearance on feature images  
  - Configuration:  
    - Splits title into lines with max ~29 characters each  
    - Saves wrapped title as `wrappedTitle` in JSON for image editing node  
  - Inputs: OpenAI generated title  
  - Outputs: Items with wrapped title  
  - Edge cases: Very long titles, titles with special characters  

---

#### 1.4 Feature Image Preparation

**Overview:**  
Fetches a base feature image, adds formatted title text for SEO and marketing, and prepares the image for upload to WordPress.

**Nodes Involved:**  
- GET image  
- Edit Feature Image  

**Node Details:**

- **GET image**  
  - Type: HTTP Request  
  - Role: Downloads a remote base feature image (static Cloudinary URL) for editing  
  - Configuration: GET request, expects binary file response  
  - Inputs: Output from Send to Wordpress2 node (post details)  
  - Outputs: Binary image data to Edit Feature Image  
  - Edge cases: HTTP errors, missing image, network failures  

- **Edit Feature Image**  
  - Type: Edit Image  
  - Role: Adds the wrapped title text onto the downloaded image with specified font, size, color, and position  
  - Configuration:  
    - Text: wrappedTitle from previous code node  
    - Font: Verdana Bold Italic  
    - Font size: 120  
    - Font color: #FD5444 (red)  
    - Position Y: 850px  
    - Line length: 180px  
    - Output: PNG format, quality 100  
    - Filename: Blog post title  
  - Inputs: Binary image  
  - Outputs: Edited image binary to Upload feature image  
  - Edge cases: Font file missing, large text causing overflow, image format issues  

---

#### 1.5 WordPress Post Creation

**Overview:**  
Uploads the edited feature image, creates the blog post as a draft with the generated content, and adds the excerpt.

**Nodes Involved:**  
- Send to Wordpress2  
- Upload feature image  
- Added feature image meta information  
- Post feature image meta information  
- Added post excerpt  

**Node Details:**

- **Send to Wordpress2**  
  - Type: WordPress node  
  - Role: Creates a new WordPress post draft with title, slug, author, category, tags, and content (HTML article)  
  - Configuration:  
    - Title: AI-generated title  
    - Slug: combination of website slug and AI-generated slug  
    - Status: draft  
    - Author and Category IDs from Edit Fields  
    - Content: HTML article from Code in JavaScript  
  - Inputs: HTML article from Code node  
  - Outputs: Post creation response with post ID  
  - Edge cases: WordPress authentication failure, API errors, content size limits  

- **Upload feature image**  
  - Type: HTTP Request  
  - Role: Uploads the edited feature image to WordPress media library via REST API  
  - Configuration:  
    - POST to `/wp-json/wp/v2/media`  
    - Sends binary image data with appropriate headers (Content-Disposition, Content-Type)  
    - Auth: WordPress API credentials  
  - Inputs: Edited image binary  
  - Outputs: Media upload response with media ID  
  - Edge cases: Upload failure, auth errors, file size limits  

- **Added feature image meta information**  
  - Type: HTTP Request  
  - Role: Adds SEO metadata (title, slug, alt text, caption, description) to the uploaded media item  
  - Configuration:  
    - POST to `/wp-json/wp/v2/media/{media_id}`  
    - Meta fields set from AI-generated content (title, slug, alt text referencing blog title)  
  - Inputs: Media upload response  
  - Outputs: Updated media info  
  - Edge cases: API errors, missing media ID  

- **Post feature image meta information**  
  - Type: HTTP Request  
  - Role: Links the uploaded media as the featured image of the WordPress post  
  - Configuration:  
    - POST to `/wp-json/wp/v2/posts/{post_id}`  
    - Sets `featured_media` to media ID from previous step  
  - Inputs: Post creation response and updated media info  
  - Outputs: Updated post info  
  - Edge cases: Post or media ID mismatch, API errors  

- **Added post excerpt**  
  - Type: HTTP Request  
  - Role: Adds a short SEO excerpt (Google excerpt) to the WordPress post  
  - Configuration:  
    - POST to `/wp-json/wp/v2/posts/{post_id}`  
    - Sets excerpt field with AI-generated Google excerpt  
  - Inputs: Post creation response  
  - Outputs: Updated post info  
  - Edge cases: API errors, excerpt length limits  

---

#### 1.6 SEO Metadata and Image Meta

**Overview:**  
Adds comprehensive Yoast SEO metadata fields to the WordPress post, including meta titles, descriptions, OpenGraph, and Twitter card data, referencing the uploaded feature image.

**Nodes Involved:**  
- Added meta title, meta description and meta image for post  

**Node Details:**

- **Added meta title, meta description and meta image for post**  
  - Type: HTTP Request  
  - Role: Updates the WordPress post with full SEO metadata compatible with Yoast SEO plugin  
  - Configuration:  
    - PUT to `/wp-json/wp/v2/posts/{post_id}`  
    - JSON body includes `_yoast_wpseo_title`, `_yoast_wpseo_metadesc`, `_yoast_wpseo_opengraph-*`, `_yoast_wpseo_twitter-*` fields  
    - Uses AI-generated meta titles and descriptions and media ID for images  
  - Inputs: Post update responses (post ID, media ID)  
  - Outputs: Finalized post metadata update  
  - Edge cases: API errors, conflicting plugin settings  

---

#### 1.7 Post Rendering and Notification

**Overview:**  
Builds a styled HTML layout for the blog post content and sends an email notification to the admin with post details after draft creation.

**Nodes Involved:**  
- Design the post  
- Send a Email to admin  

**Node Details:**

- **Design the post**  
  - Type: HTML  
  - Role: Wraps the article content and title inside a styled HTML template for WordPress formatting  
  - Configuration:  
    - HTML template with `<html>`, `<head>`, and `<body>` tags  
    - Includes CSS styling for container, headings, and text colors  
    - Inserts AI-generated title and article content dynamically  
  - Inputs: AI article content and post info  
  - Outputs: Formatted HTML string for potential use or review  
  - Edge cases: Template errors, missing dynamic data  

- **Send a Email to admin**  
  - Type: Gmail (email node)  
  - Role: Sends an email notification to an admin email address from the spreadsheet confirming draft creation  
  - Configuration:  
    - Recipient: email from spreadsheet field `confirmation email after a blog post is created`  
    - Subject: "Your Blog Post Has Been Successfully Created"  
    - Body: Includes blog title, scheduled publish date, and instructions to review in WordPress dashboard  
  - Inputs: AI content and scheduling data  
  - Outputs: Email sent confirmation  
  - Edge cases: Email sending failure, incorrect email address  

---

### 3. Summary Table

| Node Name                               | Node Type             | Functional Role                                             | Input Node(s)                              | Output Node(s)                                  | Sticky Note                                                                                                         |
|----------------------------------------|-----------------------|-------------------------------------------------------------|--------------------------------------------|-------------------------------------------------|--------------------------------------------------------------------------------------------------------------------|
| Schedule Trigger1                      | Schedule Trigger      | Daily trigger at 4 AM                                        | None                                       | Get post information from Spreadsheet           | ## STEP 1: 🟦 Trigger & Spreadsheet Input<br>Retrieves spreadsheet data, checks if today is the publish date, and prepares all fields needed for the article generation. |
| Get post information from Spreadsheet | Google Sheets         | Retrieves blog post data from Google Sheet                   | Schedule Trigger1                          | Compare date of spreadsheet and todays          | See above                                                                                                          |
| Compare date of spreadsheet and todays| If                    | Filters posts scheduled for today's date                     | Get post information from Spreadsheet      | Edit Fields                                     | See above                                                                                                          |
| Edit Fields                           | Set                   | Normalizes and prepares all fields for article generation     | Compare date of spreadsheet and todays     | GET information from sitemap                      | See above                                                                                                          |
| GET information from sitemap          | HTTP Request          | Downloads WordPress sitemap XML                              | Edit Fields                               | XML                                             | ## STEP 2:  🟨 Sitemap Fetching and Internal Link Collector<br>Downloads sitemap and extracts URLs for internal linking.|
| XML                                  | XML Parser            | Parses sitemap XML into JSON                                 | GET information from sitemap               | Split Out                                       | See above                                                                                                          |
| Split Out                            | Split Out             | Splits URL array into individual items                       | XML                                        | Aggregate                                       | See above                                                                                                          |
| Aggregate                           | Aggregate             | Aggregates URLs into a list for internal linking             | Split Out                                  | Generate all the necessary information of post  | See above                                                                                                          |
| Generate all the necessary information of post | OpenAI (Langchain)     | Generates structured blog content with AI                    | Aggregate                                  | Code in JavaScript                              | ## STEP 3:  🟩 AI Content Generation and WordPress Publisher<br>Generates structured article content using keywords.|
| Code in JavaScript                   | Code (JavaScript)     | Converts AI JSON content into final HTML article             | Generate all the necessary information of post | Send to Wordpress2                              | See above                                                                                                          |
| Code in JavaScript1                  | Code (JavaScript)     | Wraps title text for feature image formatting                | GET image                                  | Edit Feature Image                              | ## STEP 4:  🟩 GET image and editImage<br>Fetch Base Feature Image and create SEO-Friendly Feature Image             |
| Send to Wordpress2                  | WordPress             | Creates draft post on WordPress                              | Code in JavaScript                         | GET image                                       | ## STEP 3 continued                                                                                                 |
| GET image                           | HTTP Request          | Downloads base feature image                                 | Send to Wordpress2                         | Code in JavaScript1                             | See above                                                                                                          |
| Edit Feature Image                  | Edit Image            | Adds wrapped title text on feature image                      | Code in JavaScript1                        | Upload feature image                            | See above                                                                                                          |
| Upload feature image                | HTTP Request          | Uploads edited feature image to WordPress media library      | Edit Feature Image                         | Added feature image meta information            | ## STEP 5:  🟧 Upload Generated Featured Image and Add SEO-Friendly Image Meta                                     |
| Added feature image meta information | HTTP Request          | Adds SEO metadata to uploaded feature image                  | Upload feature image                       | Design the post                                 | See above                                                                                                          |
| Design the post                    | HTML                  | Wraps content in styled HTML template                        | Added feature image meta information      | Added post excerpt                              | ## STEP 6:  🟪 HTML Post Template Renderer<br>Builds blog post HTML using AI-generated content                        |
| Added post excerpt                 | HTTP Request          | Adds SEO excerpt to WordPress post                           | Design the post                           | Post feature image meta information             | ## STEP 7:  🟪 Post Excerpt Builder and Featured Image Meta Writer<br>Creates summary and adds SEO metadata          |
| Post feature image meta information | HTTP Request          | Sets uploaded image as featured image for post              | Added post excerpt                        | Added meta title, meta description and meta image for post | See above                                                                                                          |
| Added meta title, meta description and meta image for post | HTTP Request          | Adds Yoast SEO metadata fields to post                       | Post feature image meta information       | Send a Email to admin                           | ## STEP 7:  🟧 SEO Meta Writer<br>Adds Yoast SEO metadata to WordPress post                                           |
| Send a Email to admin             | Gmail                 | Sends notification email about draft creation               | Added meta title, meta description and meta image for post | None                                            | ## STEP 8:  🟩 Email Node – Send Draft Notification<br>Sends email to admin confirming draft creation                 |
| Sticky Note4                      | Sticky Note           | Documentation on workflow overview and setup                | None                                       | None                                            | # How it works... (full content in overview)                                                                       |
| Sticky Note5                      | Sticky Note           | Step 1 explanation                                           | None                                       | None                                            | ## STEP 1: 🟦 Trigger & Spreadsheet Input                                                                           |
| Sticky Note6                      | Sticky Note           | Step 2 explanation                                           | None                                       | None                                            | ## STEP 2:  🟨 Sitemap Fetching and Internal Link Collector                                                          |
| Sticky Note7                      | Sticky Note           | Step 3 explanation                                           | None                                       | None                                            | ## STEP 3:  🟩 AI Content Generation and WordPress Publisher                                                         |
| Sticky Note8                      | Sticky Note           | Step 4 explanation                                           | None                                       | None                                            | ## STEP 4:  🟩 GET image and editImage                                                                               |
| Sticky Note9                      | Sticky Note           | Step 5 explanation                                           | None                                       | None                                            | ## STEP 5:  🟧 Upload Generated Featured Image and Add SEO-Friendly Image Meta                                       |
| Sticky Note10                     | Sticky Note           | Step 6 explanation                                           | None                                       | None                                            | ## STEP 6:  🟪 HTML Post Template Renderer                                                                           |
| Sticky Note11                     | Sticky Note           | Step 7 explanation                                           | None                                       | None                                            | ## STEP 7:  🟪 Post Excerpt Builder and Featured Image Meta Writer                                                    |
| Sticky Note12                     | Sticky Note           | Step 7 SEO Metadata explanation                             | None                                       | None                                            | ## STEP 7:  🟧 SEO Meta Writer                                                                                        |
| Sticky Note13                     | Sticky Note           | Step 8 Email notification explanation                       | None                                       | None                                            | ## STEP 8:  🟩 Email Node – Send Draft Notification                                                                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger**  
   - Node: Schedule Trigger  
   - Set to trigger daily at 4:00 AM  

2. **Add Google Sheets Node**  
   - Node: Google Sheets  
   - Credentials: Connect Google Sheets credentials  
   - Operation: Read from sheet "Sheet1" in document ID `1GXRVPjsGPK5prQVvZUjg1fHKInvjqbEdAW09uCcq8sY`  
   - Purpose: Retrieve blog post data (publish date, keywords, categories, links, etc.)  

3. **Add If Node to Compare Dates**  
   - Node: If  
   - Condition: Check if "Date to be published" field (trimmed) contains the current date in ISO format derived from schedule trigger timestamp  
   - Connect Google Sheets output to If node  

4. **Add Set Node to Edit Fields**  
   - Node: Set  
   - Map all necessary fields from the filtered spreadsheet row into named fields for downstream use:  
     - Date to be published, Primary keyword, Secondary keyword, Service areas, Product Category, Internal Links, External Links, Word Count, Chapters, Style, Category ID, Author ID, Website, Call to Action URL, Customer Offer, Logo URL, Service page URL, confirmation email after post created  
   - Connect If node's True output to this node  

5. **Add HTTP Request Node to Get Sitemap**  
   - Node: HTTP Request  
   - Method: GET  
   - URL: `={{ $json.Website }}/post-sitemap.xml` (dynamic from edited fields)  

6. **Add XML Node**  
   - Node: XML  
   - Input: HTTP Request output (sitemap XML)  
   - Parses XML to JSON  

7. **Add Split Out Node**  
   - Node: Split Out  
   - Field to split: `urlset.url` (array of URLs)  

8. **Add Aggregate Node**  
   - Node: Aggregate  
   - Aggregate field: `loc`  
   - Rename aggregated field to "Blog URL List"  

9. **Add OpenAI Node (Langchain)**  
   - Node: OpenAI (Langchain)  
   - Model: GPT-4o-mini (or desired model)  
   - Prompt: As detailed in original workflow, build full article JSON including title, chapters, introduction, conclusion, FAQs, metadata, and linking instructions  
   - Input: Aggregated blog URLs and edited fields for keywords, style, offers, links, chapters count, etc.  
   - Credentials: OpenAI API credentials  

10. **Add JavaScript Code Node to Build HTML Article**  
    - Node: Code (JavaScript)  
    - Script: Extract introduction, chapters, conclusion, and FAQ from OpenAI JSON and format as HTML string with headings and paragraphs  

11. **Add WordPress Node to Create Draft Post**  
    - Node: WordPress  
    - Operation: Create post  
    - Parameters: Title, slug, content (HTML from code node), author ID, category ID, tags, status "draft"  
    - Credentials: WordPress API (Application Password recommended)  

12. **Add HTTP Request Node to Download Base Feature Image**  
    - Node: HTTP Request  
    - Method: GET  
    - URL: Base image URL (e.g. Cloudinary static image)  
    - Response: Binary file  

13. **Add JavaScript Code Node to Wrap Title Text**  
    - Node: Code (JavaScript)  
    - Script: Wrap title text to lines ~29 characters long for image overlay  

14. **Add Edit Image Node**  
    - Node: Edit Image  
    - Operation: Add text overlay  
    - Text: Wrapped title from previous code node  
    - Font: Verdana Bold Italic  
    - Font Size: 120  
    - Font Color: #FD5444  
    - Position Y: 850  

15. **Add HTTP Request Node to Upload Edited Feature Image**  
    - Node: HTTP Request  
    - Method: POST  
    - URL: `{{Website}}/wp-json/wp/v2/media`  
    - Payload: Binary image data with headers for content disposition and content type  
    - Auth: WordPress API credentials  

16. **Add HTTP Request Node to Add Feature Image Meta Information**  
    - Node: HTTP Request  
    - Method: POST  
    - URL: `/wp-json/wp/v2/media/{{media_id}}`  
    - Body parameters: title, slug, alt_text, caption, description from AI generated content  

17. **Add HTTP Request Node to Assign Feature Image to Post**  
    - Node: HTTP Request  
    - Method: POST  
    - URL: `/wp-json/wp/v2/posts/{{post_id}}`  
    - Body parameter: `featured_media` set to uploaded media ID  

18. **Add HTTP Request Node to Add Post Excerpt**  
    - Node: HTTP Request  
    - Method: POST  
    - URL: `/wp-json/wp/v2/posts/{{post_id}}`  
    - Body parameter: excerpt from AI generated Google excerpt  

19. **Add HTTP Request Node to Add SEO Metadata (Yoast)**  
    - Node: HTTP Request  
    - Method: PUT  
    - URL: `/wp-json/wp/v2/posts/{{post_id}}`  
    - JSON body: Includes Yoast SEO fields (_yoast_wpseo_title, _yoast_wpseo_metadesc, OpenGraph, Twitter card fields) populated with AI metadata and media URLs  

20. **Add HTML Node to Build Styled Post Template**  
    - Node: HTML  
    - Template: Wrap title and article content inside styled HTML and CSS  

21. **Add Gmail Node to Send Notification Email**  
    - Node: Gmail  
    - Recipient: Email from spreadsheet  
    - Subject: "Your Blog Post Has Been Successfully Created"  
    - Message: Includes post title, scheduled publish date, and WordPress dashboard instructions  
    - Credentials: Gmail OAuth2 or appropriate email service  

22. **Connect Nodes Sequentially**  
    - Follow dependency chains as per the original workflow connections  
    - Ensure error handling and credential configuration for Google Sheets, OpenAI, WordPress, and Gmail nodes  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                | Context or Link                                                                                 |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| This workflow automates scheduled WordPress publishing using content stored in a Google Sheet. It generates full SEO-friendly articles with AI assistance, internal linking based on sitemap URLs, and feature image creation. It finally sends an email notification after draft creation. Setup requires Google Sheets, OpenAI, WordPress API credentials, and Gmail or SMTP for email notifications. Adjust prompts and spreadsheet fields to suit your blog structure.                                                                    | Workflow overview and setup instructions (Sticky Note4 content)                                |
| OpenAI model used: GPT-4o-mini. You can update to a different model by changing the model ID in the "Generate all the necessary information of post" node.                                                                                                                                                                                                                                                                                                                                                                                                | OpenAI Node configuration                                                                       |
| WordPress credentials should use Application Passwords for REST API authentication for better security and ease.                                                                                                                                                                                                                                                                                                                                                                                                                                         | WordPress nodes notes                                                                          |
| The sitemap URL must be your WordPress site's post sitemap, usually found at `/post-sitemap.xml`. Adjust the URL in the HTTP Request node accordingly.                                                                                                                                                                                                                                                                                                                                                                                                   | Sitemap fetching node notes                                                                    |
| The feature image base URL is a static Cloudinary image; replace it with your own base image URL as needed.                                                                                                                                                                                                                                                                                                                                                                                                                                               | GET image node configuration                                                                  |
| The email notification is sent via Gmail node; ensure Gmail OAuth2 credentials or SMTP credentials are correctly configured.                                                                                                                                                                                                                                                                                                                                                                                                                             | Gmail node configuration                                                                      |
| The workflow uses Yoast SEO meta fields for enhanced SEO metadata; ensure Yoast SEO plugin is installed and active on your WordPress site to utilize these fields effectively.                                                                                                                                                                                                                                                                                                                                                                             | SEO metadata node notes                                                                       |
| Code nodes contain custom JavaScript to transform AI JSON output into HTML and to format title text for image overlay; review and adjust if your content structure or style requirements differ.                                                                                                                                                                                                                                                                                                                                                             | Code nodes explanations                                                                       |

---

**Disclaimer:** The text provided is exclusively sourced from an automated n8n workflow crafted for lawful content automation. It adheres strictly to current content policies and contains no illegal, offensive, or protected material. All handled data is legal and public.