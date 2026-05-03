Generate SEO-Optimized WordPress Blogs with Gemini, Tavily & Human Review

https://n8nworkflows.xyz/workflows/generate-seo-optimized-wordpress-blogs-with-gemini--tavily---human-review-8356


# Generate SEO-Optimized WordPress Blogs with Gemini, Tavily & Human Review

### 1. Workflow Overview

This workflow automates the generation, review, and publishing of SEO-optimized blog posts on a WordPress website. It is tailored for digital agencies, bloggers, and businesses that want to maintain a fresh flow of engaging content aligned with their services and trending tech topics.

The workflow includes these major logical blocks:

- **1.1 Scheduled Trigger & Topic Generation:** Automatically triggers at a scheduled interval to generate fresh blog topic ideas using Google Gemini LLM, focused on the agency's niche.

- **1.2 Research & Article Selection:** Uses the Tavily API to fetch recent articles based on the generated search query, then sends a Telegram message for human review to select the article for rewriting.

- **1.3 AI-Powered Content Rewriting:** Passes the selected article to Gemini LLM for rewriting into an SEO-optimized, factually accurate blog post, including metadata and category decisions.

- **1.4 Category Management:** Checks if the suggested category exists in WordPress; creates it if not.

- **1.5 Image Generation & Upload:** Generates a featured image using Gemini or OpenAI based on the AI-generated image prompt, then uploads it to WordPress.

- **1.6 WordPress Post Creation & Enhancement:** Creates the blog post on WordPress with the content, excerpt, category, featured image, and changes the author.

- **1.7 Audit & Notification:** Logs the published post details into a Google Sheet and notifies via Telegram.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger & Topic Generation

- **Overview:**  
  This block triggers the workflow every 4 days at 20:00, then generates a trending, service-aligned blog topic using Google Gemini LLM. The output includes a blog topic and a Tavily-friendly search query.

- **Nodes Involved:**  
  - Schedule Trigger  
  - Google Gemini Chat Model  
  - Structured Output Parser  
  - Generate the Topic using LLM

- **Node Details:**  

  - **Schedule Trigger**  
    - Type: Schedule node  
    - Configured to run every 4 days at 20:00 hour  
    - Initiates the workflow execution  
    - Input: None  
    - Output: Triggers downstream nodes  
    - Failure Modes: Scheduler misconfiguration, system downtime

  - **Google Gemini Chat Model**  
    - Type: AI language model node (Google Gemini)  
    - Uses Gemini API credential  
    - Generates topic ideas based on a prompt describing agency services and requirements  
    - Outputs raw text to the Structured Output Parser  
    - Failure: API quota, auth errors, network issues

  - **Structured Output Parser**  
    - Type: JSON output parser for LLM responses  
    - Parses Gemini’s JSON-formatted response into structured fields: `topic`, `search_query`  
    - Failure: Invalid JSON output from LLM

  - **Generate the Topic using LLM**  
    - Type: Langchain chain LLM node  
    - Contains the detailed prompt instructing the LLM to produce topic and search query JSON  
    - Has output parser enabled to ensure structured JSON output  
    - Connects to Tavily search for article retrieval

---

#### 1.2 Research & Article Selection

- **Overview:**  
  Uses the Tavily Search API to fetch 4 recent articles based on the AI-generated search query, extracts URLs, and sends a Telegram message to the user for selecting the best article.

- **Nodes Involved:**  
  - Get Articles on Topics (Tavily API)  
  - Structure Tavily Results  
  - URLs Extractor (Code)  
  - Confirm Article to Confirm (Telegram)  
  - If Article Not None

- **Node Details:**  

  - **Get Articles on Topics**  
    - Type: HTTP Request  
    - POST to Tavily API with search query, max 4 results, last week’s articles including raw content  
    - Authorization header with Tavily API key  
    - Outputs JSON results with article data  
    - Failures: API key invalid, rate limits, network issues

  - **Structure Tavily Results**  
    - Type: Set node  
    - Extracts and stores `results`, `query`, `request_id` from Tavily response for further processing  
    - Input: Tavily API response  
    - Output: Structured data for URL extraction

  - **URLs Extractor**  
    - Type: Code node (JavaScript)  
    - Extracts URLs from Tavily results array for user display  
    - Output: `{ urls: [...] }` JSON with article URLs  
    - Errors: If `results` missing or malformed

  - **Confirm Article to Confirm**  
    - Type: Telegram node with sendAndWait operation  
    - Sends a message listing URLs with dropdown for user to select article number or "None"  
    - Waits for user response asynchronously  
    - Input: URLs from previous node  
    - Failures: Telegram API errors, user timeout, malformed input

  - **If Article Not None**  
    - Type: Conditional node  
    - Checks if user selection is not "None"  
    - Routes flow accordingly to continue or restart topic generation  
    - Failures: Incorrect or missing field, expression errors

---

#### 1.3 AI-Powered Content Rewriting

- **Overview:**  
  This block structures selected article data and WordPress categories, then passes the article and categories to Gemini LLM for rewriting into a polished, SEO-optimized blog post in clean HTML with metadata.

- **Nodes Involved:**  
  - Structure Selected Post  
  - Get Existing Categories (WordPress)  
  - Aggregate Categories  
  - Write Content for the Post (Gemini LLM)  
  - Structured Output Parser1  
  - Structure Post Content

- **Node Details:**  

  - **Structure Selected Post**  
    - Type: Set node  
    - Assigns selected article data and existing categories to variables for prompt context  
    - Input: Tavily results and categories from WordPress  
    - Output: Prepared JSON for content rewriting prompt

  - **Get Existing Categories**  
    - Type: HTTP Request  
    - GET request from WordPress REST API to fetch all categories  
    - Credential: WordPress API  
    - Failure: Authentication, API downtime

  - **Aggregate Categories**  
    - Type: Aggregate node  
    - Extracts id and name fields from categories array for prompt simplification  
    - Output: List of category names for AI input

  - **Write Content for the Post**  
    - Type: Langchain chain LLM node (Google Gemini)  
    - Receives selected article content and categories as prompt context  
    - Requests SEO-optimized blog content rewrite with title, excerpt, keywords, image prompt, category selection  
    - Output: JSON with blog structure and metadata  
    - Retries on failure enabled  
    - Failures: API errors, invalid JSON output

  - **Structured Output Parser1**  
    - Type: JSON output parser  
    - Parses the rewritten blog post JSON output for further processing

  - **Structure Post Content**  
    - Type: Set node  
    - Extracts fields like title, excerpt, keywords, image prompt, content, category flags into workflow variables

---

#### 1.4 Category Management

- **Overview:**  
  Verifies if the suggested WordPress blog category exists. If not, creates a new category before proceeding.

- **Nodes Involved:**  
  - If1 (Condition)  
  - Create Category in Wordpress

- **Node Details:**  

  - **If1**  
    - Type: If node  
    - Checks `isExistingCategory` boolean flag from AI output  
    - Routes to create category or directly proceed to image generation based on condition

  - **Create Category in Wordpress**  
    - Type: HTTP Request  
    - POST to WordPress REST API to create a new category with the suggested name  
    - Credential: WordPress API  
    - Failure: Permission denied, duplicate category error

---

#### 1.5 Image Generation & Upload

- **Overview:**  
  Generates a featured image for the blog post using the AI-generated image prompt, either through Google Gemini or OpenAI DALL·E models, then uploads it to WordPress media library.

- **Nodes Involved:**  
  - Generate an Image Gemini  
  - Upload Media to Wordpress  
  - Generate an Image OpenAI (optional branch)

- **Node Details:**  

  - **Generate an Image Gemini**  
    - Type: Langchain Google Gemini node (image generation)  
    - Uses image prompt from AI content rewriting step  
    - Model: "imagen-4.0-generate-001"  
    - Output: Image binary data for upload  
    - Failure: API limits, generation errors

  - **Upload Media to Wordpress**  
    - Type: HTTP Request  
    - POST media endpoint of WordPress to upload image binary as featured media  
    - Uses WordPress API credential  
    - Sets headers for file name and MIME type dynamically  
    - Output: JSON with media ID for post association  
    - Failure: Upload errors, file size limits

  - **Generate an Image OpenAI**  
    - Type: Langchain OpenAI node (image generation)  
    - Alternative image generation method (not connected in main flow)  
    - Parameters: 1792x1024, vivid style, standard quality  
    - Failure: API errors, quota

---

#### 1.6 WordPress Post Creation & Enhancement

- **Overview:**  
  Publishes the final blog post on WordPress with the structured content, sets excerpt, assigns featured media, and changes the post author.

- **Nodes Involved:**  
  - Create a post (WordPress node)  
  - Add Excerpt on the Post  
  - Add the Featured Media on Post  
  - Change the Author

- **Node Details:**  

  - **Create a post**  
    - Type: WordPress node (native)  
    - Publishes post with title, HTML content, selected categories, and published status  
    - Inputs: Structured post content, filtered category IDs  
    - Output: WordPress post JSON including post ID and slug  
    - Failure: Auth errors, invalid data

  - **Add Excerpt on the Post**  
    - Type: HTTP Request (POST)  
    - Updates post excerpt (meta description) via WordPress REST API  
    - Uses post ID from created post  
    - Failure: API update errors

  - **Add the Featured Media on Post**  
    - Type: HTTP Request (POST)  
    - Associates uploaded media ID as the featured image of the post  
    - Uses post ID and media ID  
    - Failure: Media association errors

  - **Change the Author**  
    - Type: HTTP Request (POST)  
    - Updates the post author to user ID 1 (likely admin)  
    - Uses post ID  
    - Failure: Permission errors

---

#### 1.7 Audit & Notification

- **Overview:**  
  Logs details of the published post into a Google Sheet for auditing and sends a Telegram notification with post info and audit sheet link.

- **Nodes Involved:**  
  - Get Formatted Date & Time (Code)  
  - Structure for the Post Content  
  - Append Post Data in the Sheet (Google Sheets)  
  - Final Telegram Message

- **Node Details:**  

  - **Get Formatted Date & Time**  
    - Type: Code node (JavaScript)  
    - Generates current date/time formatted in Indian Standard Time (IST)  
    - Output: Formatted timestamp string

  - **Structure for the Post Content**  
    - Type: Set node  
    - Compiles published post data (slug, link, title, category, timestamp, featured image URL) into a structured JSON object for logging and messaging

  - **Append Post Data in the Sheet**  
    - Type: Google Sheets node  
    - Appends new row with post data to a specified sheet in Google Sheets (identified by document and sheet ID)  
    - Credential: Google Sheets OAuth2  
    - Failure: API quota, permission errors

  - **Final Telegram Message**  
    - Type: Telegram node (send message)  
    - Sends a message to a Telegram channel/group with blog post details and Google Sheets link for audit  
    - Credential: Telegram API  
    - Failure: Telegram API issues

---

### 3. Summary Table

| Node Name                   | Node Type                               | Functional Role                         | Input Node(s)                     | Output Node(s)                      | Sticky Note                                                                                                                                            |
|-----------------------------|---------------------------------------|---------------------------------------|----------------------------------|-----------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------|
| Schedule Trigger            | Schedule Trigger                      | Initiate workflow every 4 days        | None                             | Generate the Topic using LLM        | See "Auto-Blogger for WordPress" note for overall workflow description                                                                                |
| Google Gemini Chat Model    | Langchain Gemini LLM                  | Generate topic idea                   | Schedule Trigger                 | Generate the Topic using LLM        | See "Topic Generation" note                                                                                                                          |
| Structured Output Parser    | Output Parser                        | Parse LLM JSON output                  | Google Gemini Chat Model         | Generate the Topic using LLM        |                                                                                                                                                       |
| Generate the Topic using LLM| Chain LLM                           | Generate topic and tavily query       | Structured Output Parser         | Get Articles on Topics              | See "Topic Generation" note                                                                                                                          |
| Get Articles on Topics      | HTTP Request (Tavily API)             | Fetch latest articles for query       | Generate the Topic using LLM     | Structure Tavily Results            |                                                                                                                                                       |
| Structure Tavily Results    | Set                                  | Structure Tavily API response         | Get Articles on Topics           | URLs Extractor                     |                                                                                                                                                       |
| URLs Extractor              | Code                                 | Extract URLs from Tavily results      | Structure Tavily Results         | Confirm Article to Confirm          | See "Telegram Confirmation" note                                                                                                                     |
| Confirm Article to Confirm  | Telegram                             | Human selects article via Telegram    | URLs Extractor                  | If Article Not None                | See "Telegram Confirmation" note                                                                                                                     |
| If Article Not None         | If                                   | Check user selection not "None"       | Confirm Article to Confirm       | Get Existing Categories / Generate the Topic using LLM |                                                                                                                                                       |
| Get Existing Categories     | HTTP Request (WordPress API)          | Fetch WordPress categories            | If Article Not None              | Aggregate Categories               |                                                                                                                                                       |
| Aggregate Categories        | Aggregate                            | Extract category id and name          | Get Existing Categories          | Structure Selected Post             |                                                                                                                                                       |
| Structure Selected Post     | Set                                  | Prepare article and categories        | Aggregate Categories, If Article Not None | Write Content for the Post          | See "AI Content Rewriting" note                                                                                                                      |
| Write Content for the Post  | Chain LLM (Gemini)                   | Rewrite article with SEO optimization | Structure Selected Post          | Structured Output Parser1          | See "AI Content Rewriting" note                                                                                                                      |
| Structured Output Parser1   | Output Parser                        | Parse rewritten blog content JSON     | Write Content for the Post       | Structure Post Content             |                                                                                                                                                       |
| Structure Post Content      | Set                                  | Extract blog post fields for WP       | Structured Output Parser1        | If1                              |                                                                                                                                                       |
| If1                        | If                                   | Check if category exists               | Structure Post Content           | Create Category in Wordpress / Generate an Image Gemini |                                                                                                                                                       |
| Create Category in Wordpress| HTTP Request (WordPress API)          | Create new category if not existing   | If1                            | Generate an Image Gemini           |                                                                                                                                                       |
| Generate an Image Gemini    | Langchain Gemini Image Generation    | Generate featured image                | If1 (existing category branch) / Create Category in Wordpress | Upload Media to Wordpress          | See "AI Featured Image Generation" note                                                                                                              |
| Upload Media to Wordpress   | HTTP Request (WordPress API)          | Upload featured image to WordPress    | Generate an Image Gemini         | Create a post                     |                                                                                                                                                       |
| Create a post               | WordPress Node                      | Publish blog post                      | Upload Media to Wordpress        | Add Excerpt on the Post            | See "WordPress Publishing" note                                                                                                                      |
| Add Excerpt on the Post     | HTTP Request (WordPress API)          | Add meta description (excerpt)        | Create a post                   | Add the Featured Media on Post     | See "WordPress Publishing" note                                                                                                                      |
| Add the Featured Media on Post | HTTP Request (WordPress API)       | Attach featured image to post          | Add Excerpt on the Post          | Change the Author                 | See "WordPress Publishing" note                                                                                                                      |
| Change the Author           | HTTP Request (WordPress API)          | Set post author to admin (user ID 1)  | Add the Featured Media on Post   | Get Formatted Date & Time          | See "WordPress Publishing" note                                                                                                                      |
| Get Formatted Date & Time   | Code                                 | Generate IST timestamp                 | Change the Author                | Structure for the Post Content     |                                                                                                                                                       |
| Structure for the Post Content | Set                              | Prepare final post details             | Get Formatted Date & Time         | Append Post Data in the Sheet      |                                                                                                                                                       |
| Append Post Data in the Sheet | Google Sheets                      | Log post metadata in audit sheet      | Structure for the Post Content   | Final Telegram Message             | See "Telegram Notification" note                                                                                                                     |
| Final Telegram Message      | Telegram                             | Notify channel/group of new post      | Append Post Data in the Sheet    | None                             | See "Telegram Notification" note                                                                                                                     |
| Generate an Image OpenAI    | Langchain OpenAI Image Generation    | Alternative image generation (not connected) | None                         | None                             |                                                                                                                                                       |
| Get All the Users           | HTTP Request (WordPress API)          | (Unused in the main flow)              | None                           | None                             |                                                                                                                                                       |
| Sticky Notes               | Sticky Note                         | Workflow explanation and instructions | None                           | None                             | Various notes describing workflow blocks, usage instructions, and feature highlights                                                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger**  
   - Type: Schedule Trigger  
   - Set interval: Every 4 days, trigger at 20:00 (8 PM)  
   - No credentials required

2. **Add Google Gemini Chat Model node to generate topic**  
   - Type: Langchain Google Gemini LLM  
   - Use Gemini API credentials  
   - Prompt: Provide detailed instructions to generate a blog topic and Tavily search query in valid JSON format (see node “Generate the Topic using LLM”)  
   - Connect Schedule Trigger output to this node’s input

3. **Add a Structured Output Parser node**  
   - Configure with JSON schema example: `{"topic": "string", "search_query": "string"}`  
   - Connect Gemini node output to parser input

4. **Add a Chain LLM node ("Generate the Topic using LLM")**  
   - Type: Langchain Chain LLM  
   - Paste the detailed prompt text instructing Gemini to generate topic and search query JSON  
   - Enable output parser with the above Structured Output Parser  
   - Connect Structured Output Parser output into this node

5. **Add HTTP Request node to call Tavily API ("Get Articles on Topics")**  
   - URL: `https://api.tavily.com/search`  
   - Method: POST  
   - Body Parameters:  
     - `query` = `{{$json.output.search_query}}`  
     - `max_results` = 4  
     - `time_range` = "week"  
     - `include_raw_content` = "text"  
   - Header: Authorization Bearer with Tavily API key  
   - Connect output of "Generate the Topic using LLM" to this node

6. **Add a Set node ("Structure Tavily Results")**  
   - Assign variables `results`, `query`, and `request_id` from Tavily API response  
   - Connect Tavily HTTP response to this node

7. **Add a Code node ("URLs Extractor")**  
   - JavaScript code to extract URLs from `results` array  
   - Output JSON: `{ urls: [...] }`  
   - Connect "Structure Tavily Results" output here

8. **Add Telegram node ("Confirm Article to Confirm")**  
   - Operation: sendAndWait  
   - Chat ID: Your Telegram channel/group  
   - Message: List URLs with selectable dropdown (options 1-4 + None)  
   - Wait for user response  
   - Connect "URLs Extractor" output to this node

9. **Add If node ("If Article Not None")**  
   - Condition: Check if user response is not "None"  
   - If true: continue; if false: loop back or end workflow

10. **Add HTTP Request node ("Get Existing Categories")**  
    - Method: GET  
    - URL: WordPress categories endpoint `/wp-json/wp/v2/categories`  
    - Use WordPress API credentials  
    - Connect from If node True branch

11. **Add Aggregate node ("Aggregate Categories")**  
    - Extract `id` and `name` from categories array  
    - Connect "Get Existing Categories" output here

12. **Add Set node ("Structure Selected Post")**  
    - Assign selected article from Tavily results based on user selection  
    - Include categories array and category names string  
    - Connect from Aggregate node output and If node True branch

13. **Add Chain LLM node ("Write Content for the Post")**  
    - Receive selected article content and categories string  
    - Prompt Gemini to rewrite article into SEO optimized blog post with metadata and HTML content  
    - Enable output parser with JSON schema including title, excerpt, keywords, image_prompt, category flags, content  
    - Connect "Structure Selected Post" output here

14. **Add Structured Output Parser ("Structured Output Parser1")**  
    - Parse JSON output from "Write Content for the Post" node  
    - Connect output of "Write Content for the Post" here

15. **Add Set node ("Structure Post Content")**  
    - Extract fields: title, excerpt, keywords, image_prompt, content, isExistingCategory, categoryName  
    - Connect "Structured Output Parser1" output here

16. **Add If node ("If1")**  
    - Check if `isExistingCategory` is true  
    - If true: go to image generation  
    - If false: create category first

17. **Add HTTP Request ("Create Category in Wordpress")**  
    - POST to WordPress categories endpoint `/wp-json/wp/v2/categories`  
    - Body: `name` = categoryName from AI output  
    - Use WordPress credentials  
    - Connect from If1 false branch

18. **Add Langchain Gemini Image Generation node ("Generate an Image Gemini")**  
    - Prompt: `image_prompt` from AI content rewriting output  
    - Model: `imagen-4.0-generate-001`  
    - Connect from If1 true branch, and also from "Create Category in Wordpress" node success

19. **Add HTTP Request node ("Upload Media to Wordpress")**  
    - POST to WordPress media endpoint `/wp-json/wp/v2/media`  
    - Send binary image data from the Gemini image generation node  
    - Set headers `content-disposition` and `content-type` dynamically from binary data  
    - Use WordPress credentials  
    - Connect from "Generate an Image Gemini"

20. **Add WordPress node ("Create a post")**  
    - Title and content from "Structure Post Content"  
    - Categories filtered by matching category name to IDs from WordPress  
    - Status: publish  
    - Connect from "Upload Media to Wordpress"

21. **Add HTTP Request ("Add Excerpt on the Post")**  
    - POST to `/wp-json/wp/v2/posts/{post_id}`  
    - Body: excerpt from "Structure Post Content"  
    - Connect from "Create a post"

22. **Add HTTP Request ("Add the Featured Media on Post")**  
    - POST to `/wp-json/wp/v2/posts/{post_id}`  
    - Body: featured_media = uploaded media ID  
    - Connect from "Add Excerpt on the Post"

23. **Add HTTP Request ("Change the Author")**  
    - POST to `/wp-json/wp/v2/posts/{post_id}`  
    - Body: author = 1 (admin user)  
    - Connect from "Add the Featured Media on Post"

24. **Add Code node ("Get Formatted Date & Time")**  
    - Output current date/time in IST formatted string  
    - Connect from "Change the Author"

25. **Add Set node ("Structure for the Post Content")**  
    - Prepare slug, link, title, categoryName, timestamp, featured image source_url for logging and messaging  
    - Connect from "Get Formatted Date & Time"

26. **Add Google Sheets node ("Append Post Data in the Sheet")**  
    - Append post metadata to audit Google Sheet  
    - Set columns: Link, Slug, Title, Category, Timestamp, Featured Image  
    - Use Google Sheets OAuth2 credentials  
    - Connect from "Structure for the Post Content"

27. **Add Telegram node ("Final Telegram Message")**  
    - Send message with post details and Google Sheet link to Telegram group/channel  
    - Use Telegram API credentials  
    - Connect from "Append Post Data in the Sheet"

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                               | Context or Link                                                                                                                   |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------|
| Auto-Blogger for Wordpress: Scheduled fully automated SEO blog post generation with Gemini, Tavily, Telegram human review, image generation, WordPress publishing, and audit in Sheets.      | Workflow Sticky Note                                                                                                              |
| Topic Generation: Gemini LLM generates unique blog topics aligned with agency services and current tech trends; creates Tavily-friendly search queries.                                   | Workflow Sticky Note                                                                                                              |
| Telegram Confirmation: Human-in-the-loop step via Telegram to select the best article for rewriting ensures quality control.                                                               | Workflow Sticky Note                                                                                                              |
| AI Content Rewriting: Gemini LLM rewrites selected article into SEO-optimized, factually accurate blog post with metadata and HTML content.                                               | Workflow Sticky Note                                                                                                              |
| AI Featured Image Generation: Gemini or OpenAI generates a unique featured image based on AI prompt for blog posts.                                                                        | Workflow Sticky Note                                                                                                              |
| WordPress Publishing: Posts created with full content, excerpt, categories, tags, featured media, and author assignment.                                                                  | Workflow Sticky Note                                                                                                              |
| Telegram Notification: Sends final post details and link to audit Google Sheet to Telegram channel/group.                                                                                  | Workflow Sticky Note                                                                                                              |
| How to Use: Connect required credentials (WordPress API, Gemini/OpenAI APIs, Google Sheets OAuth, Telegram Bot, Tavily API Key), configure schedule, customize prompt, activate workflow. | Workflow Sticky Note                                                                                                              |
| Google Sheets Audit Link: https://docs.google.com/spreadsheets/d/1aaSBeDXtU9n4wmgoH-mn3ett54vCDCMwxzlyhJCiPqA/edit?usp=sharing                                                               | Included in Telegram message for audit trail                                                                                      |

---

**Disclaimer:**  
The provided text is exclusively derived from an n8n automation workflow. All data processed is legal and public, and no illegal, offensive, or protected content is included. The workflow complies with content policies in effect.