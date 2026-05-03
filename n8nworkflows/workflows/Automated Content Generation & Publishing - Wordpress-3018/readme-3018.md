Automated Content Generation & Publishing - Wordpress

https://n8nworkflows.xyz/workflows/automated-content-generation---publishing---wordpress-3018


# Automated Content Generation & Publishing - Wordpress

### 1. Workflow Overview

This workflow automates the generation, enrichment, and publishing of blog posts on a self-hosted WordPress website. It is designed primarily for bloggers, marketers, and businesses aiming to streamline content creation and scheduling with AI assistance and automated media integration.

The workflow is logically divided into the following blocks:

- **1.1 Trigger & Scheduling**  
  Handles workflow initiation either on a schedule or manually, including randomized delay to simulate natural posting times.

- **1.2 AI Content Generation**  
  Uses OpenAI’s GPT-4o model to generate SEO-optimized, structured blog articles with embedded keywords for image search.

- **1.3 Data Persistence**  
  Saves generated content and keywords to a Google Sheet for record-keeping and potential manual review.

- **1.4 Automated Image Retrieval**  
  Queries the Pexels API to fetch relevant stock images based on keywords extracted from the AI-generated content.

- **1.5 WordPress Post Creation**  
  Publishes the post to WordPress via its REST API, embedding the retrieved image and content, and setting metadata such as status and featured image.

---

### 2. Block-by-Block Analysis

#### 1.1 Trigger & Scheduling

**Overview:**  
This block initiates the workflow either on a predefined schedule or manually, then applies a randomized delay (0-6 hours) before proceeding to content generation to avoid predictable posting patterns.

**Nodes Involved:**  
- 1. Auto Start (Schedule Trigger) [Disabled]  
- 2. When clicking ‘Test workflow’ (Manual Trigger) [Disabled]  
- 3. Schedule Your Posts (Schedule Trigger)  
- Processing Delay (Code)  
- Random Wait (Wait)

**Node Details:**

- **1. Auto Start**  
  - Type: Schedule Trigger  
  - Role: Automatically triggers workflow every 1 minute (disabled by default)  
  - Configuration: Interval set to 1 minute  
  - Inputs: None  
  - Outputs: Connects to no nodes (disabled)  
  - Edge Cases: Disabled; no runtime impact unless enabled.

- **2. When clicking ‘Test workflow’**  
  - Type: Manual Trigger  
  - Role: Allows manual triggering for testing purposes (disabled by default)  
  - Inputs: None  
  - Outputs: None (disabled)  
  - Edge Cases: Disabled; no runtime impact unless enabled.

- **3. Schedule Your Posts**  
  - Type: Schedule Trigger  
  - Role: Triggers workflow on specific days of the week (Tuesday, Thursday, Sunday) at noon (12:00)  
  - Inputs: None  
  - Outputs: Connects to Processing Delay node  
  - Edge Cases: Timezone considerations (Asia/Taipei); ensure server time sync.

- **Processing Delay**  
  - Type: Code (JavaScript)  
  - Role: Generates a random delay between 0 and 6 hours (in milliseconds)  
  - Configuration: Uses Math.random() to calculate delay; outputs delay in ms, minutes, and hours  
  - Inputs: From Schedule Your Posts  
  - Outputs: Connects to Random Wait node  
  - Edge Cases: None significant; random delay always generated.

- **Random Wait**  
  - Type: Wait  
  - Role: Pauses workflow execution for the computed delay duration  
  - Configuration: Wait time set dynamically from Processing Delay output (`delay` in seconds)  
  - Inputs: From Processing Delay  
  - Outputs: Connects to Generate AI Content node  
  - Edge Cases: Long delays may cause workflow timeouts if n8n instance has execution limits.

---

#### 1.2 AI Content Generation

**Overview:**  
Generates a full-length, SEO-optimized blog article with structured headings and extracts 3-5 relevant keywords for image search using OpenAI GPT-4o.

**Nodes Involved:**  
- Generate AI Content

**Node Details:**

- **Generate AI Content**  
  - Type: OpenAI (LangChain)  
  - Role: Sends a detailed prompt to GPT-4o to generate article JSON including title, content (HTML formatted), and keywords array  
  - Configuration:  
    - Model: GPT-4o  
    - Prompt: Includes instructions for SEO-friendly formatting, HTML structure, keyword extraction, and article sections (Introduction, Core Content, Conclusion)  
    - Output: JSON with fields `title`, `content`, and `keywords` (array)  
  - Inputs: From Random Wait  
  - Outputs: Connects to Save to Sheet node  
  - Credentials: Requires OpenAI API key configured in n8n  
  - Edge Cases:  
    - API rate limits or quota exceeded  
    - Prompt formatting errors causing invalid JSON output  
    - Network timeouts or API errors

---

#### 1.3 Data Persistence

**Overview:**  
Stores generated article data (title, content, keywords) into a Google Sheet for tracking, auditing, or manual editing.

**Nodes Involved:**  
- Save to Sheet

**Node Details:**

- **Save to Sheet**  
  - Type: Google Sheets  
  - Role: Appends a new row with article title, content, and concatenated keywords (joined by '+')  
  - Configuration:  
    - Operation: Append  
    - Sheet: "Sheet1"  
    - Columns mapped:  
      - title → `message.content.title` from AI output  
      - content → `message.content.content` from AI output  
      - Image search keyword → keywords array joined by '+'  
  - Inputs: From Generate AI Content  
  - Outputs: Connects to Automated Image Retrieval from Pexels node  
  - Credentials: Requires Google Sheets OAuth2 credentials  
  - Edge Cases:  
    - Google API quota limits  
    - Sheet access permission errors  
    - Invalid data causing append failure

---

#### 1.4 Automated Image Retrieval

**Overview:**  
Fetches a relevant stock image from Pexels API using the keywords extracted from the AI-generated content to visually enrich the blog post.

**Nodes Involved:**  
- Automated Image Retrieval from Pexels

**Node Details:**

- **Automated Image Retrieval from Pexels**  
  - Type: HTTP Request  
  - Role: Queries Pexels API to retrieve one landscape-oriented image matching the keyword string  
  - Configuration:  
    - URL: `https://api.pexels.com/v1/search?per_page=1&orientation=landscape&query={{ $json["Image search keyword"] }}`  
    - Headers: Authorization with Pexels API key, Content-Type application/json  
    - Query Parameters: `query` dynamically set from Google Sheet keywords  
  - Inputs: From Save to Sheet  
  - Outputs: Connects to Create posts on Wordpress node  
  - Edge Cases:  
    - API key invalid or quota exceeded  
    - No images found for given keywords (empty photos array)  
    - Network errors or timeouts

---

#### 1.5 WordPress Post Creation

**Overview:**  
Publishes the blog post to WordPress via REST API, embedding the retrieved image as inline content and setting post metadata such as status and title.

**Nodes Involved:**  
- Create posts on Wordpress

**Node Details:**

- **Create posts on Wordpress**  
  - Type: WordPress node (REST API integration)  
  - Role: Creates and publishes a new post on WordPress with title, content (including embedded image), and status set to "publish"  
  - Configuration:  
    - Title: Taken from Google Sheet `title` field  
    - Content: HTML including an `<img>` tag with the first Pexels image URL (landscape) plus the article content from Google Sheet  
    - Status: "publish" (immediate publishing)  
  - Inputs: From Automated Image Retrieval from Pexels  
  - Outputs: None (end of workflow)  
  - Credentials: WordPress API credentials configured in n8n  
  - Edge Cases:  
    - WordPress REST API authentication errors  
    - Invalid content causing post creation failure  
    - FIFU plugin recommended for featured image from URL support  
    - Network or server errors

---

### 3. Summary Table

| Node Name                         | Node Type                  | Functional Role                       | Input Node(s)                   | Output Node(s)                      | Sticky Note                                                                                         |
|----------------------------------|----------------------------|------------------------------------|--------------------------------|-----------------------------------|---------------------------------------------------------------------------------------------------|
| 1. Auto Start                    | Schedule Trigger           | Auto trigger every minute (disabled) | None                           | None (disabled)                   |                                                                                                   |
| 2. When clicking ‘Test workflow’ | Manual Trigger             | Manual trigger for testing (disabled) | None                           | None (disabled)                   |                                                                                                   |
| 3. Schedule Your Posts           | Schedule Trigger           | Scheduled workflow trigger (Tue, Thu, Sun at 12:00) | None                           | Processing Delay                  |                                                                                                   |
| Processing Delay                | Code                      | Generates random delay (0-6h)       | 3. Schedule Your Posts          | Random Wait                      |                                                                                                   |
| Random Wait                    | Wait                      | Pauses workflow for randomized delay | Processing Delay                | Generate AI Content              |                                                                                                   |
| Generate AI Content             | OpenAI (LangChain)        | Generates SEO-optimized article and keywords | Random Wait                    | Save to Sheet                   | Sticky Note1, Sticky Note2: Detailed prompt instructions and formatting requirements               |
| Save to Sheet                  | Google Sheets             | Saves article data to Google Sheet  | Generate AI Content             | Automated Image Retrieval from Pexels | Sticky Note3: Google Sheet column setup and mapping instructions                                   |
| Automated Image Retrieval from Pexels | HTTP Request              | Fetches relevant image from Pexels API | Save to Sheet                  | Create posts on Wordpress       | Sticky Note7: Reminder to add API credential                                                      |
| Create posts on Wordpress       | WordPress                 | Publishes post with content and image | Automated Image Retrieval from Pexels | None                          | Sticky Note4: Recommendation to install FIFU plugin for featured image support; Sticky Note5,6,7: Add API credentials |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Nodes:**

   - Add a **Schedule Trigger** node named "3. Schedule Your Posts"  
     - Set schedule to trigger weekly on Tuesday, Thursday, and Sunday at 12:00 (noon)  
     - Timezone: Asia/Taipei (or your preferred timezone)  

   - Optionally add **Schedule Trigger** "1. Auto Start" (disabled) for 1-minute interval  
   - Optionally add **Manual Trigger** "2. When clicking ‘Test workflow’" (disabled) for manual testing  

2. **Add Processing Delay Node:**

   - Add a **Code** node named "Processing Delay"  
   - Paste the following JavaScript code:  
     ```js
     const delay = Math.floor(Math.random() * (6 * 60 * 60 * 1000)); // 0-6 hours in ms
     return {
       json: {
         delay: delay,
         delay_minutes: Math.round(delay / 60000),
         delay_hours: (delay / 3600000).toFixed(2)
       }
     };
     ```  
   - Connect "3. Schedule Your Posts" output to this node  

3. **Add Random Wait Node:**

   - Add a **Wait** node named "Random Wait"  
   - Set "Wait Time" to expression: `{{$json["delay"] / 1000}}` (convert ms to seconds)  
   - Connect "Processing Delay" output to this node  

4. **Add AI Content Generation Node:**

   - Add an **OpenAI (LangChain)** node named "Generate AI Content"  
   - Select model: GPT-4o (or preferred GPT-4 variant)  
   - Configure messages with the detailed prompt (see Sticky Note2 content) instructing:  
     - Generate SEO-optimized article with HTML formatting  
     - Extract 3-5 image search keywords  
     - Output JSON with fields: `title`, `content`, `keywords` (array)  
   - Enable JSON output  
   - Connect "Random Wait" output to this node  
   - Configure OpenAI API credentials  

5. **Add Google Sheets Node:**

   - Add a **Google Sheets** node named "Save to Sheet"  
   - Operation: Append  
   - Document: Select or enter your Google Sheet URL  
   - Sheet Name: "Sheet1"  
   - Map columns:  
     - `title` → `{{$json["message"]["content"]["title"]}}`  
     - `content` → `{{$json["message"]["content"]["content"]}}`  
     - `Image search keyword` → `{{$json["message"]["content"]["keywords"].join("+")}}`  
   - Connect "Generate AI Content" output to this node  
   - Configure Google Sheets OAuth2 credentials  

6. **Add HTTP Request Node for Pexels:**

   - Add an **HTTP Request** node named "Automated Image Retrieval from Pexels"  
   - Method: GET  
   - URL: `https://api.pexels.com/v1/search?per_page=1&orientation=landscape&query={{ $json["Image search keyword"] }}`  
   - Headers:  
     - Authorization: Your Pexels API key  
     - Content-Type: application/json  
   - Connect "Save to Sheet" output to this node  

7. **Add WordPress Node:**

   - Add a **WordPress** node named "Create posts on Wordpress"  
   - Operation: Create Post  
   - Title: `{{$node["Save to Sheet"].json["title"]}}`  
   - Content:  
     ```html
     <img src="{{$node["Automated Image Retrieval from Pexels"].json["photos"][0].src.landscape}}" alt="image text" style="width:100%; height:auto;"><br><br>
     <br><br>
     {{$node["Save to Sheet"].json["content"]}}
     ```  
   - Status: Publish  
   - Connect "Automated Image Retrieval from Pexels" output to this node  
   - Configure WordPress API credentials (REST API enabled on your WordPress site)  

8. **Optional: Add Sticky Notes**

   - Add sticky notes with the provided content for documentation and instructions within the workflow canvas.

9. **Workflow Settings**

   - Set timezone to Asia/Taipei or your preferred timezone  
   - Enable manual execution saving for debugging  
   - Adjust execution timeout as needed  

---

### 5. General Notes & Resources

| Note Content                                                                                         | Context or Link                                                                                          |
|----------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| Highly recommended to install the WordPress plugin "Featured Image from URL (FIFU)" and enable "Auto > Set Featured Media Automatically from Content" for featured image support. | Sticky Note4                                                                                             |
| Use the detailed prompt provided in Sticky Note2 to ensure AI generates well-structured, SEO-friendly articles with embedded keywords for image search. | Sticky Note2                                                                                             |
| Google Sheet must have columns named exactly: "title", "content", and "Image search keyword" for correct data mapping. | Sticky Note3                                                                                             |
| Add your API credentials for OpenAI, Google Sheets, Pexels, and WordPress in the respective nodes before running. | Sticky Note5, Sticky Note6, Sticky Note7                                                                |
| This workflow supports randomized delays before publishing to simulate natural posting behavior and avoid bulk posting detection. | Sticky Note (top left)                                                                                    |
| For more information on WordPress REST API and authentication, refer to official WordPress developer documentation. | https://developer.wordpress.org/rest-api/                                                               |
| Pexels API documentation for image search and usage limits: https://www.pexels.com/api/documentation/ | https://www.pexels.com/api/documentation/                                                                |
| OpenAI API documentation for prompt design and rate limits: https://platform.openai.com/docs/       | https://platform.openai.com/docs/                                                                         |

---

This documentation provides a complete, structured understanding of the "Automated Content Generation & Publishing - Wordpress" workflow, enabling reproduction, modification, and troubleshooting by advanced users or automation agents.