Transform Blog Posts to Multi-Platform Content with GPT-4o, Unsplash and Airtable

https://n8nworkflows.xyz/workflows/transform-blog-posts-to-multi-platform-content-with-gpt-4o--unsplash-and-airtable-9164


# Transform Blog Posts to Multi-Platform Content with GPT-4o, Unsplash and Airtable

### 1. Workflow Overview

This workflow automates the transformation of newly published blog posts into multi-platform social media and email content, leveraging AI (GPT-4o), Unsplash image search, Airtable storage, and Slack notifications. The main goal is to repurpose long-form blog content into tailored formats for LinkedIn, Twitter, Instagram, email newsletters, and short-form video scripts, streamlining content marketing efforts with minimal manual input.

Logical blocks in the workflow:

- **1.1 Input Reception:** Detect new blog posts via RSS feed and fetch their full HTML content.
- **1.2 Content Cleaning & Extraction:** Clean the HTML, extract raw text, and prepare a content snippet for AI processing.
- **1.3 AI Content Repurposing:** Use GPT-4o to generate social media posts, email snippets, and video scripts in a structured JSON format.
- **1.4 AI Output Parsing & Structuring:** Parse and structure the AI JSON output, merging with original metadata.
- **1.5 Image Suggestions:** Query Unsplash for relevant images based on the blog title and attach image suggestions.
- **1.6 Notification & Storage:** Notify the content team on Slack and save all repurposed content and metadata to Airtable.
- **1.7 Optional Auto-Posting:** Disabled by default, nodes to post directly to LinkedIn and Twitter.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:** Detects new blog posts by reading a configured RSS feed and fetches the full blog post HTML content.
- **Nodes Involved:**  
  - New Blog Post4  
  - Fetch Full Content4  

- **Node Details:**

  - **New Blog Post4**  
    - Type: RSS Feed Read  
    - Role: Polls RSS feed URL for new blog post entries.  
    - Configuration: URL parameter set to `https://yourblog.com/feed` (to be replaced with user’s blog RSS).  
    - Input: None, trigger node.  
    - Output: Provides metadata such as title, link, creator, pubDate.  
    - Edge cases: RSS feed URL misconfiguration, network timeout, invalid RSS XML.

  - **Fetch Full Content4**  
    - Type: HTTP Request  
    - Role: Fetches full HTML content of the blog post from the link provided by RSS.  
    - Configuration: URL dynamically set to the blog post link from RSS (`={{ $json.link }}`), expects plain text response.  
    - Input: Output from New Blog Post4.  
    - Output: Raw HTML content as text.  
    - Edge cases: HTTP errors (404, 500), redirects, slow response, malformed HTML.

---

#### 1.2 Content Cleaning & Extraction

- **Overview:** Cleans HTML content to extract readable text and prepares a preview snippet for AI input.  
- **Nodes Involved:**  
  - Clean & Extract Content4  

- **Node Details:**

  - **Clean & Extract Content4**  
    - Type: Code (JavaScript)  
    - Role: Strips scripts/styles and HTML tags from fetched content, extracts metadata from RSS feed, truncates to 2000 characters for AI.  
    - Configuration: Custom JS code that removes HTML tags and extracts excerpt and full content.  
    - Inputs: Raw HTML from Fetch Full Content4, metadata from New Blog Post4.  
    - Outputs: JSON containing title, url, author, published_date, content (preview), full_content, excerpt.  
    - Edge cases: HTML parsing errors if blog content is malformed; missing metadata fields fallback to 'Unknown'.

---

#### 1.3 AI Content Repurposing

- **Overview:** Sends cleaned blog content to GPT-4o to generate repurposed content for multiple platforms, following strict format and style guidelines.  
- **Nodes Involved:**  
  - AI Content Repurposing Chain4  
  - OpenAI Chat Model5  
  - Structured Output Parser5  

- **Node Details:**

  - **AI Content Repurposing Chain4**  
    - Type: LangChain LLM Chain  
    - Role: Defines prompt template instructing GPT-4o to create structured content for LinkedIn, Twitter, Instagram, email, video scripts.  
    - Configuration: Prompt includes blog title, content snippet, URL, and specifies JSON output format with detailed requirements per platform (character limits, tone, hashtags, CTAs).  
    - Input: Extracted content from Clean & Extract Content4.  
    - Output: Raw text JSON response from OpenAI.  
    - Edge cases: Prompt failures, token limits, bad AI output formats.

  - **OpenAI Chat Model5**  
    - Type: LangChain OpenAI Chat Model  
    - Role: Calls GPT-4o with configured parameters (max tokens 3000, temperature 0.7), expects JSON object response.  
    - Configuration: GPT-4o model, temperature balancing creativity and guidance, response as JSON object.  
    - Input: Prompt from AI Content Repurposing Chain4.  
    - Output: AI-generated JSON content.  
    - Edge cases: API rate limits, auth failures, incomplete responses.

  - **Structured Output Parser5**  
    - Type: LangChain Output Parser (Structured)  
    - Role: Validates and auto-fixes AI JSON output according to predefined schema specifying required fields and types for each content format.  
    - Configuration: Manual JSON schema for LinkedIn post, Twitter thread, Instagram caption, email snippet, and video script.  
    - Input: Raw AI JSON text from OpenAI Chat Model5.  
    - Output: Parsed and validated structured JSON.  
    - Edge cases: Schema validation errors, autoFix may not handle all malformed outputs.

---

#### 1.4 AI Output Parsing & Structuring

- **Overview:** Merges AI-generated repurposed content with original blog metadata and prepares final JSON object for downstream use.  
- **Nodes Involved:**  
  - Structure Output4  

- **Node Details:**

  - **Structure Output4**  
    - Type: Code (JavaScript)  
    - Role: Parses the AI-generated JSON, adds original title, URL, published date, repurposed date timestamp, and sets content status.  
    - Configuration: Parses JSON string from AI, merges with Clean & Extract Content4 metadata.  
    - Input: Parsed AI output from Structured Output Parser5.  
    - Output: Consolidated JSON with all content forms and metadata.  
    - Edge cases: JSON parse errors if AI output is invalid.

---

#### 1.5 Image Suggestions

- **Overview:** Searches Unsplash for relevant images based on the blog title and appends image suggestions with metadata.  
- **Nodes Involved:**  
  - Fetch Images (Unsplash)4  
  - Add Image Suggestions4  

- **Node Details:**

  - **Fetch Images (Unsplash)4**  
    - Type: HTTP Request  
    - Role: Queries Unsplash API for photos matching first three words of original blog title, limited to 3 results.  
    - Configuration: URL dynamically constructed using first 3 words of original_title, authentication via HTTP header with Unsplash API key.  
    - Input: Output from Structure Output4.  
    - Output: JSON with Unsplash search results.  
    - Edge cases: API quota limits, auth errors, no results found.

  - **Add Image Suggestions4**  
    - Type: Code (JavaScript)  
    - Role: Maps Unsplash results into simplified image objects with URLs, thumbnails, alt text, photographer, download link.  
    - Configuration: Extracts top 3 images, adds them as suggested_images array to existing content JSON.  
    - Input: Unsplash API results.  
    - Output: Enriched JSON including suggested_images.  
    - Edge cases: Empty or malformed Unsplash results.

---

#### 1.6 Notification & Storage

- **Overview:** Sends a Slack notification to the content team with content summaries and saves the repurposed content into Airtable for review and tracking.  
- **Nodes Involved:**  
  - Notify Content Team4  
  - Save to Content Library4  

- **Node Details:**

  - **Notify Content Team4**  
    - Type: Slack  
    - Role: Posts a formatted message to a configured Slack channel with blog title, URL, publish date, content formats ready, preview snippet, and image suggestion count.  
    - Configuration: Uses Slack OAuth2 credentials, channel ID must be replaced with target content channel. Message uses expressions to dynamically insert content data.  
    - Input: Output from Add Image Suggestions4.  
    - Output: Confirmation of Slack message sent.  
    - Edge cases: Slack auth errors, invalid channel ID, message formatting issues.

  - **Save to Content Library4**  
    - Type: Airtable  
    - Role: Creates a new record in a specified Airtable base and table, mapping all repurposed content and metadata to appropriate columns.  
    - Configuration: Base ID and Table ID must be replaced by user. Columns mapped include original title, URLs, publication date, all repurposed text fields, hashtags, status set to "Pending Review".  
    - Input: Output from Notify Content Team4.  
    - Output: Airtable record creation confirmation.  
    - Edge cases: Airtable API limits, credential/auth failures, column mapping errors.

---

#### 1.7 Optional Auto-Posting

- **Overview:** Disabled by default. Nodes to automatically post generated LinkedIn and Twitter content using respective APIs.  
- **Nodes Involved:**  
  - Post to LinkedIn (Optional)4  
  - Post to Twitter (Optional)4  

- **Node Details:**

  - **Post to LinkedIn (Optional)4**  
    - Type: LinkedIn  
    - Role: Posts the LinkedIn post text and hashtags along with a "Read more" link to an organization’s LinkedIn feed.  
    - Configuration: Disabled by default, requires organization ID and OAuth2 credentials setup.  
    - Input: Output from Save to Content Library4.  
    - Output: LinkedIn post confirmation.  
    - Edge cases: LinkedIn API permission errors, invalid org ID, post content length limits.

  - **Post to Twitter (Optional)4**  
    - Type: Twitter  
    - Role: Posts the first tweet of the Twitter thread to Twitter.  
    - Configuration: Disabled by default, requires Twitter OAuth2 credentials. Currently only posts first tweet; threading logic not implemented.  
    - Input: Output from Save to Content Library4.  
    - Output: Twitter post confirmation.  
    - Edge cases: Twitter API rate limits, auth errors, tweet length limits.

---

### 3. Summary Table

| Node Name                  | Node Type                       | Functional Role                          | Input Node(s)             | Output Node(s)                  | Sticky Note                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   |
|----------------------------|--------------------------------|----------------------------------------|---------------------------|--------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Sticky Note3               | Sticky Note                    | Documentation and setup instructions   | None                      | None                           | 🤖 AI Content Repurposing Engine Setup instructions for RSS, Slack, Airtable, credentials, optional auto-posting, and workflow overview.                                                                                                                                                                                                                                                                                                                                                                       |
| New Blog Post4             | RSS Feed Read                  | Detect new blog posts                   | None                      | Fetch Full Content4             |                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              |
| Fetch Full Content4        | HTTP Request                  | Fetch full HTML content of blog post   | New Blog Post4            | Clean & Extract Content4        |                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              |
| Clean & Extract Content4   | Code (JavaScript)             | Clean HTML, extract text & metadata    | Fetch Full Content4       | AI Content Repurposing Chain4   |                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              |
| AI Content Repurposing Chain4 | LangChain Chain LLM            | Prepare AI prompt for content repurpose | Clean & Extract Content4   | OpenAI Chat Model5, Structured Output Parser5 |                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              |
| OpenAI Chat Model5         | LangChain OpenAI Chat Model   | Call GPT-4o API with prompt            | AI Content Repurposing Chain4 | Structured Output Parser5       |                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              |
| Structured Output Parser5  | LangChain Output Parser       | Validate and parse AI JSON output      | OpenAI Chat Model5        | AI Content Repurposing Chain4   |                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              |
| Structure Output4          | Code (JavaScript)             | Merge AI output with blog metadata     | Structured Output Parser5 | Fetch Images (Unsplash)4        |                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              |
| Fetch Images (Unsplash)4   | HTTP Request                  | Search Unsplash for relevant images    | Structure Output4         | Add Image Suggestions4          |                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              |
| Add Image Suggestions4     | Code (JavaScript)             | Extract and format image suggestions   | Fetch Images (Unsplash)4  | Notify Content Team4            |                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              |
| Notify Content Team4       | Slack                        | Notify team of new repurposed content  | Add Image Suggestions4    | Save to Content Library4        |                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              |
| Save to Content Library4   | Airtable                     | Save repurposed content and metadata   | Notify Content Team4      | Post to LinkedIn (Optional)4, Post to Twitter (Optional)4 |                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              |
| Post to LinkedIn (Optional)4 | LinkedIn                    | Auto-post LinkedIn content (optional) | Save to Content Library4  | None                           | Disabled by default. Requires LinkedIn organization ID and OAuth2 credentials.                                                                                                                                                                                                                                                                                                                                                                                                                               |
| Post to Twitter (Optional)4 | Twitter                     | Auto-post Twitter content (optional)   | Save to Content Library4  | None                           | Disabled by default. Requires Twitter OAuth2 credentials. Only posts first tweet of thread.                                                                                                                                                                                                                                                                                                                                                                                                                   |

---

### 4. Reproducing the Workflow from Scratch

**Step 1: Create RSS Feed Read Node**  
- Name: `New Blog Post4`  
- Type: RSS Feed Read  
- Parameters: Set URL to your blog’s RSS feed, e.g., `https://yourblog.com/feed`  
- Purpose: To trigger the workflow when a new blog post is published.

**Step 2: Create HTTP Request Node to Fetch Full Blog Content**  
- Name: `Fetch Full Content4`  
- Type: HTTP Request  
- Parameters:  
  - URL: `={{ $json.link }}` (dynamic from RSS feed)  
  - Response Format: Text  
- Connect: Output of `New Blog Post4` → Input of `Fetch Full Content4`.

**Step 3: Create Code Node to Clean and Extract Text**  
- Name: `Clean & Extract Content4`  
- Type: Code (JavaScript)  
- Parameters: Insert code to strip HTML tags, remove scripts/styles, trim to 2000 characters, and extract metadata fields (title, URL, author, date).  
- Connect: Output of `Fetch Full Content4` → Input of `Clean & Extract Content4`.

**Step 4: Setup LangChain LLM Chain to Prepare Prompt**  
- Name: `AI Content Repurposing Chain4`  
- Type: LangChain Chain LLM  
- Parameters:  
  - Define prompt template instructing GPT-4o to create LinkedIn, Twitter thread, Instagram caption, email snippet, and video script in JSON format. Use placeholders for title, content snippet, and URL.  
- Connect: Output of `Clean & Extract Content4` → Input of `AI Content Repurposing Chain4`.

**Step 5: Setup OpenAI Chat Model Node**  
- Name: `OpenAI Chat Model5`  
- Type: LangChain OpenAI Chat Model  
- Parameters:  
  - Model: `gpt-4o`  
  - Max tokens: 3000  
  - Temperature: 0.7  
  - Response format: JSON object  
- Credentials: Add OpenAI API key with GPT-4o access.  
- Connect: Output of `AI Content Repurposing Chain4` (prompt) → Input of `OpenAI Chat Model5`.

**Step 6: Setup Structured Output Parser to Validate AI Response**  
- Name: `Structured Output Parser5`  
- Type: LangChain Output Parser (Structured)  
- Parameters:  
  - Provide JSON schema defining required fields for LinkedIn post, Twitter thread, Instagram caption, email snippet, and video script.  
  - Enable auto-fix.  
- Connect: Output of `OpenAI Chat Model5` → Input of `Structured Output Parser5`.  
- Connect parser output back to `AI Content Repurposing Chain4` (for LangChain internal usage).

**Step 7: Create Code Node to Structure Final Output**  
- Name: `Structure Output4`  
- Type: Code (JavaScript)  
- Parameters: Parse AI JSON output, merge with original blog metadata (title, URL, publish date), add repurposed date timestamp and status.  
- Connect: Output of `Structured Output Parser5` → Input of `Structure Output4`.

**Step 8: Create HTTP Request Node to Fetch Unsplash Images**  
- Name: `Fetch Images (Unsplash)4`  
- Type: HTTP Request  
- Parameters:  
  - URL: `https://api.unsplash.com/search/photos?query={{ $json.original_title.split(' ').slice(0, 3).join(' ') }}&per_page=3`  
  - Authentication: HTTP Header Auth with Unsplash API key.  
- Credentials: Configure Unsplash API key under HTTP Header Auth.  
- Connect: Output of `Structure Output4` → Input of `Fetch Images (Unsplash)4`.

**Step 9: Create Code Node to Add Image Suggestions**  
- Name: `Add Image Suggestions4`  
- Type: Code (JavaScript)  
- Parameters: Map Unsplash search results to array of image metadata (URL, thumbnail, alt text, photographer, download link). Attach to content JSON.  
- Connect: Output of `Fetch Images (Unsplash)4` → Input of `Add Image Suggestions4`.

**Step 10: Create Slack Node to Notify Content Team**  
- Name: `Notify Content Team4`  
- Type: Slack  
- Parameters:  
  - Message text with dynamic content summary, previews, hashtags, and image count.  
  - Channel ID: Replace with your Slack content channel ID.  
- Credentials: Configure Slack OAuth2 credentials.  
- Connect: Output of `Add Image Suggestions4` → Input of `Notify Content Team4`.

**Step 11: Create Airtable Node to Save Content**  
- Name: `Save to Content Library4`  
- Type: Airtable  
- Parameters:  
  - Base ID and Table ID: Replace with your Airtable base and table identifiers.  
  - Map columns to fields such as Original_Title, Original_URL, Published_Date, LinkedIn_Post, Twitter_Thread, Instagram_Caption, Email_Subject, Email_Body, Video_Script, Suggested_Images, Status.  
  - Set Status default to "Pending Review".  
- Credentials: Configure Airtable API token.  
- Connect: Output of `Notify Content Team4` → Input of `Save to Content Library4`.

**Step 12: Optional Auto-Posting Nodes**  
- Create LinkedIn node `Post to LinkedIn (Optional)4`, disabled by default.  
  - Text: Combine LinkedIn post text, hashtags, and "Read more" link.  
  - Configure with LinkedIn OAuth2 and organization ID.  
- Create Twitter node `Post to Twitter (Optional)4`, disabled by default.  
  - Text: Post first tweet of Twitter thread.  
  - Configure with Twitter OAuth2 credentials.  
- Connect: Output of `Save to Content Library4` → Input of both optional posting nodes.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     | Context or Link                                                                                                                        |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------|
| Setup instructions include replacing placeholders for RSS feed URL, Slack channel ID, Airtable base/table IDs, LinkedIn organization ID, and adding credentials for OpenAI GPT-4o, Unsplash API, Slack OAuth2, Airtable token, and optionally Twitter OAuth2.                                                                                                                                                                                                                                                    | Sticky Note node labeled “Sticky Note3” at workflow start provides detailed step-by-step setup instructions and overview.          |
| AI prompt is carefully crafted to produce JSON output for five different content formats optimized by platform constraints, including character limits and tone.                                                                                                                                                                                                                                                                                                                                             | See AI Content Repurposing Chain4 node parameters.                                                                                   |
| Unsplash API requires proper HTTP header authentication with a valid API key to fetch image suggestions.                                                                                                                                                                                                                                                                                                                                                                                                         | See Fetch Images (Unsplash)4 node configuration and credentials.                                                                     |
| Slack notifications provide content previews and summary, facilitating content team review and approval before publishing.                                                                                                                                                                                                                                                                                                                                                                                     | Notify Content Team4 node message template.                                                                                          |
| Airtable is used as a centralized content library with status tracking (“Pending Review”).                                                                                                                                                                                                                                                                                                                                                                                                                         | Save to Content Library4 node with defined column mappings.                                                                          |
| Auto-posting to LinkedIn and Twitter is disabled by default for manual review workflow but can be enabled by setting credentials and toggling node activation.                                                                                                                                                                                                                                                                                                                                                  | Post to LinkedIn (Optional)4 and Post to Twitter (Optional)4 nodes.                                                                  |
| The workflow depends on stable API connections and valid credentials for OpenAI, Unsplash, Slack, Airtable, LinkedIn, and Twitter. Error handling should be added in production for API limits, auth errors, and invalid data formats.                                                                                                                                                                                                                                                                           | Best practice recommendation for production use.                                                                                    |

---

**Disclaimer:**  
The text provided is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.