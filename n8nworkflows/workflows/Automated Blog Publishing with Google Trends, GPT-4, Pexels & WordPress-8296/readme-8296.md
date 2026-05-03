Automated Blog Publishing with Google Trends, GPT-4, Pexels & WordPress

https://n8nworkflows.xyz/workflows/automated-blog-publishing-with-google-trends--gpt-4--pexels---wordpress-8296


# Automated Blog Publishing with Google Trends, GPT-4, Pexels & WordPress

### 1. Workflow Overview

This workflow, titled **"Automated Blog Publishing with Google Trends, GPT-4, Pexels & WordPress"**, automates the end-to-end process of generating SEO-optimized blog posts based on trending keywords from Google Trends, enriching content with AI-generated articles, searching for relevant images, and publishing drafts on WordPress. It is designed for content marketers, bloggers, and SEO professionals aiming to streamline content creation and publication while leveraging data-driven keyword trends and AI writing capabilities.

The workflow is logically divided into the following blocks:

- **1.1 Configuration and Triggering:** Setup of workflow parameters and recurring execution.
- **1.2 Data Acquisition and Filtering:** Fetch trending keywords from Google Trends RSS feed, parse and filter them by traffic.
- **1.3 Google Sheets Integration:** Append or update keyword data in a Google Sheet and retrieve keywords to process.
- **1.4 AI Content Generation:** Use GPT-4 via Langchain to generate SEO-friendly blog posts based on keywords and example titles.
- **1.5 Image Search and Processing:** Search Pexels for relevant images, add watermark, and upload media to WordPress.
- **1.6 WordPress Publishing:** Create draft posts, associate images as featured media, and update post metadata.
- **1.7 Data Cleanup and Status Updates:** Clean HTML content and update processing status in Google Sheets.

---

### 2. Block-by-Block Analysis

#### 1.1 Configuration and Triggering

**Overview:**  
Initial setup of minimum traffic threshold for filtering keywords and scheduling the workflow to run every 25 minutes.

**Nodes Involved:**  
- Schedule Trigger  
- Main Config

**Node Details:**

- **Schedule Trigger**  
  - Type: Schedule Trigger  
  - Role: Initiates workflow execution every 25 minutes.  
  - Configuration: Interval set to 25 minutes.  
  - Input: None (trigger node)  
  - Output: Connects to Main Config node.  
  - Edge Cases: Scheduler downtime or execution delays may cause missed runs.

- **Main Config**  
  - Type: Set  
  - Role: Defines workflow configuration parameters.  
  - Configuration: Sets numeric variable `min_traffic` to 500.  
  - Input: From Schedule Trigger node.  
  - Output: Connects to GoogleTrends HTTP Request node.  
  - Edge Cases: Incorrect or missing config values could affect filtering accuracy.

---

#### 1.2 Data Acquisition and Filtering

**Overview:**  
Fetches the Google Trends RSS feed for the US region, parses the XML response, extracts trending keywords and traffic estimates, then filters keywords based on minimum traffic.

**Nodes Involved:**  
- GoogleTrends  
- XML  
- Filter Scraped Keywords

**Node Details:**

- **GoogleTrends**  
  - Type: HTTP Request  
  - Role: Retrieves RSS feed from Google Trends (US).  
  - Configuration: GET request to `https://trends.google.it/trending/rss` with query parameter `geo=US`.  
  - Input: From Main Config.  
  - Output: Connects to XML node.  
  - Edge Cases: Network failure, API changes, or invalid geo code may cause no data or errors.  
  - Retry enabled on failure.

- **XML**  
  - Type: XML Parse  
  - Role: Converts RSS XML content into JSON for processing.  
  - Configuration: No normalization, explicit arrays disabled.  
  - Input: From GoogleTrends output.  
  - Output: Connects to Filter Scraped Keywords.  
  - Edge Cases: Malformed XML may cause parsing errors.

- **Filter Scraped Keywords**  
  - Type: Code (JavaScript)  
  - Role: Extracts keywords and approximate traffic from parsed XML, filters by `min_traffic`.  
  - Configuration: Custom JS code:  
    - Reads `min_traffic` from Main Config.  
    - Extracts up to 3 example news titles per keyword.  
    - Parses `approx_traffic` removing '+' and filtering by minimum threshold.  
    - Sorts keywords descending by traffic.  
  - Input: From XML node and Main Config via references.  
  - Output: Returns filtered, sorted keyword list to Loop Over Items node.  
  - Edge Cases: Missing traffic fields, empty results from Google Trends, or expression failures.

---

#### 1.3 Google Sheets Integration

**Overview:**  
Maintains a Google Sheet as a dynamic database for trending keywords and their processing status, appending new or updating existing rows, and retrieving keywords marked for processing.

**Nodes Involved:**  
- Loop Over Items  
- Append or update row in sheet  
- Get row in sheet

**Node Details:**

- **Loop Over Items**  
  - Type: Split In Batches  
  - Role: Processes keyword items one by one for Google Sheets operations.  
  - Configuration: Default batch size (1).  
  - Input: From Filter Scraped Keywords.  
  - Output: Connects to Append or Update Row and Get Row in sheet nodes.

- **Append or update row in sheet**  
  - Type: Google Sheets  
  - Role: Inserts or updates keyword records in Google Sheet by matching on Keyword column.  
  - Configuration:  
    - Document ID and sheet specified.  
    - Columns mapped: ID (random number), Date, Keyword, Traffic, Example Titles 1-3.  
    - Matching column: Keyword.  
  - Input: From Loop Over Items.  
  - Output: Loops back to Loop Over Items for next batch.  
  - Credentials: Google Sheets OAuth2.  
  - Edge Cases: Rate limits, authentication errors, or schema mismatches.

- **Get row in sheet**  
  - Type: Google Sheets  
  - Role: Retrieves rows where Status is "processing" (keywords ready for AI content generation).  
  - Configuration: Filter on Status column = "processing".  
  - Input: From Loop Over Items.  
  - Output: Connects to Grab one random keyword node.  
  - Credentials: Google Sheets OAuth2.  
  - Edge Cases: Empty result sets or filter misconfiguration.

---

#### 1.4 AI Content Generation

**Overview:**  
Generates a high-quality, SEO-optimized blog post based on a selected keyword and example titles using GPT-4 via Langchain. Parses structured JSON output for title, description, content, and image query.

**Nodes Involved:**  
- Grab one random keyword  
- AI Agent (Langchain Agent)  
- OpenAI Chat Model  
- Structured Output Parser  
- Cleanup the AI

**Node Details:**

- **Grab one random keyword**  
  - Type: Code (JavaScript)  
  - Role: Randomly selects one keyword marked as "processing" and with valid example titles for processing.  
  - Configuration: Filters items with Status="processing" and non-empty Keyword and Example Title 1.  
  - Input: From Get row in sheet.  
  - Output: Selected keyword to AI Agent node.  
  - Edge Cases: No keywords available results in empty output.

- **AI Agent**  
  - Type: Langchain Agent (OpenAI LLM)  
  - Role: Uses GPT-4 to generate an SEO-friendly blog article with metadata.  
  - Configuration:  
    - Custom prompt instructing the agent to write in a conversational tone, structure content with HTML tags, target 800-1200 words, and produce JSON output with `title`, `description`, `content`, and `images_query`.  
    - No images included in content.  
  - Input: From Grab one random keyword node; uses keyword and example titles.  
  - Output: Connects to Structured Output Parser node.  
  - Credentials: OpenAI API (GPT-4).  
  - Edge Cases: API rate limits, prompt misinterpretation, or output parsing failures.

- **OpenAI Chat Model**  
  - Type: Langchain OpenAI Chat Model  
  - Role: Provides GPT-4.1-mini model backend for AI Agent.  
  - Input: From AI Agent node (ai_languageModel port).  
  - Output: Back to AI Agent for response.  
  - Credentials: OpenAI API key.  
  - Edge Cases: API connectivity or quota issues.

- **Structured Output Parser**  
  - Type: Langchain Structured Output Parser  
  - Role: Parses the AI-generated JSON output strictly into expected schema.  
  - Configuration: JSON schema example provided for title, description, content, images_query.  
  - Input: From AI Agent node (ai_outputParser port).  
  - Output: Back to AI Agent node for further processing.

- **Cleanup the AI**  
  - Type: Code (JavaScript)  
  - Role: Cleans unwanted special characters from HTML content produced by AI, preserving semantic tags.  
  - Configuration: Removes em/en dashes, ellipsis, fancy quotes, zero-width spaces, collapses multiple spaces.  
  - Input: From Search Pexels Image node.  
  - Output: Connects to Create posts on Wordpress node.  
  - Edge Cases: Malformed HTML input could cause partial cleanup.

---

#### 1.5 Image Search and Processing

**Overview:**  
Searches Pexels API for an image related to the article's topic, adds a watermark, and uploads the image to WordPress media library.

**Nodes Involved:**  
- Search Pexels Image  
- Add watermark to image  
- Upload media  
- Update image metadata  
- Set image ID for the post

**Node Details:**

- **Search Pexels Image**  
  - Type: HTTP Request  
  - Role: Queries Pexels API for images based on AI-generated `images_query` string.  
  - Configuration:  
    - GET request with query parameter `query` encoded from `images_query` field; requests 1 image (`per_page=1`).  
    - Uses API key via HTTP Header Authentication credential.  
  - Input: From Update row in sheet node (after AI processing marked as done).  
  - Output: Connects to Cleanup the AI node.  
  - Edge Cases: API rate limits, no images found, authentication errors.

- **Add watermark to image**  
  - Type: HTTP Request  
  - Role: Sends the Pexels image URL to QuickChart API to add a watermark overlay image.  
  - Configuration: POST request with JSON payload including main image URL, watermark URL, and watermark ratio (0.25).  
  - Input: From Create posts on Wordpress node.  
  - Output: Connects to Upload media node.  
  - Edge Cases: API failure or incorrect image URLs.

- **Upload media**  
  - Type: HTTP Request  
  - Role: Uploads the watermarked image binary data to WordPress media library.  
  - Configuration:  
    - POST to WordPress REST API media endpoint.  
    - Sends binary data with proper Content-Disposition header including filename from `images_query`.  
    - Uses WordPress API credential.  
  - Input: From Add watermark to image node.  
  - Output: Connects to Set image ID for the post node.  
  - Edge Cases: Upload failure, authentication error, or file size limits.

- **Update image metadata**  
  - Type: HTTP Request  
  - Role: Updates the uploaded media item metadata (title, alt text, description) on WordPress.  
  - Configuration: POST request with JSON body setting title, alt text, description based on AI-generated article title and description.  
  - Input: From Set image ID for the post node.  
  - Output: None (end node in this chain).  
  - Credentials: WordPress API.  
  - Edge Cases: API errors or permission issues.

- **Set image ID for the post**  
  - Type: HTTP Request  
  - Role: Associates the uploaded image as the featured media for the WordPress post.  
  - Configuration: POST request to update the post's `featured_media` field with the media ID.  
  - Input: From Upload media node.  
  - Output: Connects to Update image metadata node.  
  - Credentials: WordPress API.  
  - Edge Cases: Post or media ID mismatch, API failure.

---

#### 1.6 WordPress Publishing

**Overview:**  
Creates a draft WordPress post with the AI-generated content and title, assigns post categories, and enables ping status.

**Nodes Involved:**  
- Create posts on Wordpress

**Node Details:**

- **Create posts on Wordpress**  
  - Type: WordPress REST API node  
  - Role: Creates a new draft blog post on WordPress site.  
  - Configuration:  
    - Title and content set from AI Agent output.  
    - Post status: draft.  
    - Categories: category ID 4.  
    - Ping status: open.  
  - Input: From Cleanup the AI node (cleaned HTML content).  
  - Output: Connects to Add watermark to image node.  
  - Credentials: WordPress API.  
  - Edge Cases: API errors, invalid category IDs, content length limits.

---

#### 1.7 Data Cleanup and Status Updates

**Overview:**  
Updates the keyword's status to "done" in Google Sheets after AI content generation, and ensures data integrity across the workflow.

**Nodes Involved:**  
- Update row in sheet

**Node Details:**

- **Update row in sheet**  
  - Type: Google Sheets  
  - Role: Updates the keyword row status to "done" to prevent re-processing.  
  - Configuration:  
    - Matches by Keyword column.  
    - Sets Status column to "done".  
  - Input: From AI Agent node (after generating content).  
  - Output: Connects to Search Pexels Image node.  
  - Credentials: Google Sheets OAuth2.  
  - Edge Cases: Race conditions if multiple workflows run concurrently, authentication errors.

---

### 3. Summary Table

| Node Name                 | Node Type                           | Functional Role                                  | Input Node(s)                  | Output Node(s)               | Sticky Note                                                                                         |
|---------------------------|-----------------------------------|-------------------------------------------------|-------------------------------|-----------------------------|---------------------------------------------------------------------------------------------------|
| Schedule Trigger          | Schedule Trigger                  | Starts workflow every 25 minutes                 | None                          | Main Config                 |                                                                                                   |
| Main Config              | Set                              | Sets config variable `min_traffic`               | Schedule Trigger              | GoogleTrends                | config                                                                                            |
| GoogleTrends             | HTTP Request                     | Fetch Google Trends RSS feed for US               | Main Config                  | XML                        |                                                                                                   |
| XML                      | XML Parse                       | Parses RSS XML to JSON                             | GoogleTrends                 | Filter Scraped Keywords     |                                                                                                   |
| Filter Scraped Keywords   | Code (JavaScript)                | Extracts keywords, filters by traffic             | XML, Main Config             | Loop Over Items             |                                                                                                   |
| Loop Over Items           | Split In Batches                 | Processes keywords in batches                      | Filter Scraped Keywords       | Append or update row in sheet, Get row in sheet |                                                                                                   |
| Append or update row in sheet | Google Sheets                 | Inserts or updates keyword data in Google Sheet   | Loop Over Items              | Loop Over Items             |                                                                                                   |
| Get row in sheet          | Google Sheets                   | Retrieves rows with Status = "processing"          | Loop Over Items              | Grab one random keyword     |                                                                                                   |
| Grab one random keyword   | Code (JavaScript)                | Selects a random keyword marked for processing     | Get row in sheet             | AI Agent                   |                                                                                                   |
| AI Agent                 | Langchain Agent (OpenAI GPT-4)  | Generates SEO-optimized blog content                | Grab one random keyword       | Update row in sheet (after AI Agent) |                                                                                                   |
| OpenAI Chat Model        | Langchain OpenAI Model           | Provides GPT-4.1-mini model for AI Agent           | AI Agent (ai_languageModel)  | AI Agent                   |                                                                                                   |
| Structured Output Parser | Langchain Output Parser Structured | Parses AI JSON output                               | AI Agent (ai_outputParser)   | AI Agent                   |                                                                                                   |
| Update row in sheet      | Google Sheets                   | Marks keyword as "done" after AI content generation | AI Agent                    | Search Pexels Image         |                                                                                                   |
| Search Pexels Image      | HTTP Request                    | Searches Pexels for related images                  | Update row in sheet          | Cleanup the AI              |                                                                                                   |
| Cleanup the AI           | Code (JavaScript)                | Cleans unwanted characters from AI-generated HTML   | Search Pexels Image          | Create posts on Wordpress   |                                                                                                   |
| Create posts on Wordpress | WordPress REST API              | Creates draft post with AI content                   | Cleanup the AI               | Add watermark to image      |                                                                                                   |
| Add watermark to image   | HTTP Request                    | Adds watermark overlay to image                      | Create posts on Wordpress    | Upload media                |                                                                                                   |
| Upload media             | HTTP Request                    | Uploads watermarked image to WordPress media        | Add watermark to image       | Set image ID for the post   |                                                                                                   |
| Set image ID for the post | HTTP Request                   | Sets uploaded image as featured media for post      | Upload media                 | Update image metadata       |                                                                                                   |
| Update image metadata    | HTTP Request                    | Updates image metadata (title, alt text, desc)      | Set image ID for the post    | None                       |                                                                                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger node**  
   - Type: Schedule Trigger  
   - Set interval to 25 minutes.

2. **Create Main Config node**  
   - Type: Set  
   - Add numeric field `min_traffic` with value `500`.  
   - Connect Schedule Trigger output to this node.

3. **Create GoogleTrends node**  
   - Type: HTTP Request  
   - Method: GET  
   - URL: `https://trends.google.it/trending/rss`  
   - Query Parameter: `geo=US`  
   - Enable retry on failure.  
   - Connect Main Config node output to this node.

4. **Create XML node**  
   - Type: XML Parse  
   - Disable normalization and explicit array.  
   - Connect GoogleTrends output to this node.

5. **Create Filter Scraped Keywords node**  
   - Type: Code (JavaScript)  
   - Paste provided JS code that extracts keywords, parses traffic numbers, filters by `min_traffic` (referencing Main Config), and sorts descending.  
   - Connect XML output to this node.

6. **Create Loop Over Items node**  
   - Type: Split In Batches  
   - Default batch size.  
   - Connect Filter Scraped Keywords output to this node.

7. **Create Append or update row in sheet node**  
   - Type: Google Sheets  
   - Operation: Append or Update  
   - Document ID: Google Sheet for keywords database  
   - Sheet: gid=0  
   - Map columns: ID (random number), Date, Keyword, Traffic, Example Title 1-3  
   - Set matching column to Keyword.  
   - Connect Loop Over Items output to this node.

8. **Connect Append or update row output back to Loop Over Items to continue processing batches.**

9. **Create Get row in sheet node**  
   - Type: Google Sheets  
   - Operation: Read Rows  
   - Filter: Status column equals "processing"  
   - Document and sheet same as above.  
   - Connect Loop Over Items output to this node.

10. **Create Grab one random keyword node**  
    - Type: Code (JavaScript)  
    - Paste code to randomly select an item from rows with Status="processing" and non-empty keywords/titles.  
    - Connect Get row in sheet output to this node.

11. **Create AI Agent node**  
    - Type: Langchain Agent  
    - Set prompt as given: instruct GPT-4 to generate SEO-friendly blog post with title, description, content, and images_query JSON output.  
    - Connect Grab one random keyword output to this node.

12. **Create OpenAI Chat Model node**  
    - Type: Langchain OpenAI Chat Model  
    - Select model: GPT-4.1-mini  
    - Connect AI Agent's `ai_languageModel` input to this node.

13. **Create Structured Output Parser node**  
    - Type: Langchain Output Parser Structured  
    - Provide JSON schema example with title, description, content, images_query.  
    - Connect AI Agent's `ai_outputParser` output to this node.

14. **Create Update row in sheet node**  
    - Type: Google Sheets  
    - Operation: Update  
    - Match on Keyword column  
    - Set Status to "done"  
    - Connect AI Agent output main to this node.

15. **Create Search Pexels Image node**  
    - Type: HTTP Request  
    - Method: GET  
    - URL: `https://api.pexels.com/v1/search?query={{encodeURIComponent($('AI Agent').item.json.output.images_query)}}&per_page=1`  
    - Authentication: HTTP Header Auth with Pexels API key  
    - Connect Update row in sheet output to this node.

16. **Create Cleanup the AI node**  
    - Type: Code (JavaScript)  
    - Paste provided JS code for cleaning unwanted characters from AI-generated HTML content.  
    - Connect Search Pexels Image output to this node.

17. **Create Create posts on Wordpress node**  
    - Type: WordPress API node  
    - Operation: Create Post  
    - Title: From AI Agent output title  
    - Content: From cleaned AI content  
    - Status: draft  
    - Categories: [4] (adjust as needed)  
    - Ping status: open  
    - Connect Cleanup the AI output to this node.

18. **Create Add watermark to image node**  
    - Type: HTTP Request  
    - Method: POST  
    - URL: `https://quickchart.io/watermark`  
    - Body parameters: mainImageUrl (from Pexels image URL), markImageUrl (watermark logo URL), markRatio 0.25  
    - Connect Create posts on Wordpress output to this node.

19. **Create Upload media node**  
    - Type: HTTP Request  
    - Method: POST  
    - URL: WordPress media endpoint (`https://clipmint.pro/wp-json/wp/v2/media`)  
    - Content-Type: binaryData  
    - Headers: Content-Disposition with filename from images_query  
    - Authentication: WordPress API credentials  
    - Connect Add watermark to image output to this node.

20. **Create Set image ID for the post node**  
    - Type: HTTP Request  
    - Method: POST  
    - URL: WordPress post endpoint to update post metadata (`.../posts/{{post_id}}`)  
    - Query parameter: featured_media = uploaded media ID  
    - Authentication: WordPress API  
    - Connect Upload media output to this node.

21. **Create Update image metadata node**  
    - Type: HTTP Request  
    - Method: POST  
    - URL: WordPress media endpoint for uploaded media ID  
    - JSON body: title, alt_text, description from AI Agent output  
    - Authentication: WordPress API  
    - Connect Set image ID for the post output to this node.

22. **Connect nodes ensuring proper data flow as described in the workflow connections.**

23. **Configure all credentials properly:**  
    - Google Sheets OAuth2  
    - OpenAI API key for GPT-4  
    - Pexels API key via HTTP Header Auth  
    - WordPress API OAuth or Application Password credentials

---

### 5. General Notes & Resources

| Note Content                                                                                                                                    | Context or Link                                   |
|-------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------|
| The AI Agent prompt is carefully crafted to produce human-like, SEO-optimized, and WordPress-ready HTML content with strict output JSON format. | AI content generation guidelines                  |
| Google Trends RSS feed is sourced from `https://trends.google.it/trending/rss` with geo set to US, which may be adapted for other regions.     | Google Trends RSS feed documentation               |
| Pexels API requires an API key sent via HTTP header authentication for image search.                                                           | https://www.pexels.com/api/documentation/          |
| The watermark is applied via QuickChart watermark API with the Clipmint logo overlay.                                                           | https://quickchart.io/documentation/watermark-api |
| The workflow updates Google Sheets to track processing status, avoiding duplicate content generation.                                           | Google Sheets API documentation                     |
| WordPress nodes use REST API endpoints for post, media creation and metadata updates, requiring proper authentication and permissions.          | WordPress REST API docs: https://developer.wordpress.org/rest-api/ |

---

**Disclaimer:**  
The provided text is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected material. All manipulated data is legal and publicly accessible.