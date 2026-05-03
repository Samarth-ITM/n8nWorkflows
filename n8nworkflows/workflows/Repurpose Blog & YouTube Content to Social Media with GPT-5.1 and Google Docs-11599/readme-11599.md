Repurpose Blog & YouTube Content to Social Media with GPT-5.1 and Google Docs

https://n8nworkflows.xyz/workflows/repurpose-blog---youtube-content-to-social-media-with-gpt-5-1-and-google-docs-11599


# Repurpose Blog & YouTube Content to Social Media with GPT-5.1 and Google Docs

### 1. Workflow Overview

This workflow is an AI-powered content repurposing engine designed to transform input content from blogs, YouTube videos, or raw text into multiple social media and marketing formats, then output the results as a formatted Google Doc. It is targeted at content creators, marketers, and social media managers who want to efficiently generate platform-optimized posts and collateral from existing content sources.

**Logical blocks of the workflow:**

- **1.1 Input Reception:** Receives user input via a form supporting three types of content sources (blog URL, YouTube URL, raw text) along with optional metadata like target audience and custom instructions.

- **1.2 Content Extraction:** Depending on the input type, fetches and extracts text content:
  - For blogs: HTTP fetch + HTML parsing and cleaning
  - For YouTube: Video ID extraction, API metadata fetch, page scraping for transcript
  - For Raw Text: Direct pass-through with minor parsing

- **1.3 AI Content Generation:** Invokes OpenAI GPT-5.1 with a detailed prompt to analyze the extracted content and create multiple repurposed content formats, including LinkedIn posts, Twitter threads, emails, quotes, and video scripts.

- **1.4 Document Formatting & Output:** Parses the AI JSON output, assembles a comprehensive Google Doc with organized sections for each content type, creates the Google Doc, writes the content, and prepares a response with the document URL.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  Captures user input via an n8n form webhook, enabling content submission in three formats plus optional target audience and instructions.

- **Nodes Involved:**  
  - Content Input Form  
  - Is Blog URL? (IF)  
  - Is YouTube URL? (IF)

- **Node Details:**  

  - **Content Input Form**  
    - Type: Form Trigger  
    - Role: Entry point to collect user input with fields for content type (dropdown), content input (textarea), target audience (optional), and additional instructions (optional).  
    - Key Parameters:  
      - Content Type options: Blog Post URL, YouTube Video URL, Raw Text / Paste Content  
      - Form path: `/content-repurpose`  
      - Required fields: Content Type, Content Input  
      - Form description clarifies purpose and credential requirements  
    - Inputs: Webhook request  
    - Outputs: Passes JSON with form data downstream  
    - Edge Cases: User submits invalid URLs or empty content; no built-in validation beyond required fields.

  - **Is Blog URL?**  
    - Type: IF  
    - Role: Checks if the content type is "Blog Post URL"  
    - Condition: `$json['Content Type'] === 'Blog Post URL'`  
    - Inputs: Content Input Form output  
    - Outputs: True branch leads to blog extraction; False leads to next IF node  
    - Edge Cases: Case sensitivity enforced; input must exactly match string.

  - **Is YouTube URL?**  
    - Type: IF  
    - Role: Checks if the content type is "YouTube Video URL"  
    - Condition: `$json['Content Type'] === 'YouTube Video URL'`  
    - Inputs: From Is Blog URL? false branch  
    - Outputs: True leads to YouTube extraction; False leads to raw text processing  
    - Edge Cases: Same as above.

---

#### 2.2 Content Extraction

- **Overview:**  
  Fetches and processes the source content according to its type, normalizing it into a unified JSON format suitable for AI processing.

- **Nodes Involved:**  
  - Fetch Blog Page  
  - Extract Blog Content  
  - Extract Video ID  
  - Get YouTube Metadata  
  - Fetch YouTube Page  
  - Process YouTube Data  
  - Process Raw Text

- **Node Details:**  

  - **Fetch Blog Page**  
    - Type: HTTP Request  
    - Role: Downloads the blog URL HTML content  
    - Config: URL from form input, 30s timeout, expects text response  
    - Inputs: From Is Blog URL? true branch  
    - Outputs: Raw HTML content  
    - Edge Cases: HTTP errors, timeouts, invalid URLs, allowed to continue on fail to avoid breaking workflow.

  - **Extract Blog Content**  
    - Type: Code  
    - Role: Parses the blog HTML to extract title, author, description, and main article text  
    - Key Logic: Uses regex to find title tags, meta description, author meta tag; cleans HTML removing scripts, nav, footer, etc.; extracts main content from `<article>`, `<main>`, or div classes typical for blogs; strips HTML tags; truncates if content > 12,000 chars; throws error if content too short.  
    - Inputs: Fetch Blog Page output and original form data for metadata fields  
    - Outputs: JSON with contentType='blog', cleaned content, metadata, word count, timestamps  
    - Edge Cases: Page structure variability, missing tags, malformed HTML, very short content, potential extraction failure.

  - **Extract Video ID**  
    - Type: Code  
    - Role: Extracts YouTube video ID from various possible YouTube URL formats  
    - Inputs: Form input JSON  
    - Outputs: videoId and metadata fields  
    - Edge Cases: Unsupported URL formats, missing video ID, malformed URLs; throws errors if ID cannot be found.

  - **Get YouTube Metadata**  
    - Type: HTTP Request  
    - Role: Calls YouTube Data API v3 to fetch video snippet, content details, and statistics  
    - Config: Queries by videoId, authenticated with YouTube OAuth2 credential  
    - Inputs: Extract Video ID output  
    - Outputs: JSON with video metadata  
    - Edge Cases: API quota limits, auth errors, network timeouts, missing video data.

  - **Fetch YouTube Page**  
    - Type: HTTP Request  
    - Role: Fetches YouTube video page HTML (to scrape captions/transcripts)  
    - Inputs: Extract Video ID output (videoId) to build URL  
    - Outputs: Raw HTML page content  
    - Edge Cases: Page structure changes, rate limiting, timeouts, missing captions.

  - **Process YouTube Data**  
    - Type: Code  
    - Role: Merges YouTube API metadata and scraped page data to create a unified content object  
    - Key Logic: Parses metadata for title, description, channel, views, likes; attempts to detect captions/transcripts from HTML using regex; builds markdown content summary; truncates if too long  
    - Inputs: Outputs from Extract Video ID, Get YouTube Metadata, Fetch YouTube Page  
    - Outputs: JSON with contentType='youtube', aggregated content, metadata, word count, timestamps  
    - Edge Cases: Missing metadata fields, no captions detected, JSON parse errors, malformed HTML.

  - **Process Raw Text**  
    - Type: Code  
    - Role: Passes raw text input through with minimal processing, extracts first line as potential title if appropriate, truncates if too long  
    - Inputs: Form input JSON  
    - Outputs: JSON with contentType='raw_text', content, title, metadata  
    - Edge Cases: Very short or malformed text, no title found.

---

#### 2.3 AI Content Generation

- **Overview:**  
  Uses OpenAI GPT-5.1 model to analyze the extracted content and generate multiple repurposed content formats optimized for social platforms and email marketing.

- **Nodes Involved:**  
  - AI Content Generator

- **Node Details:**  

  - **AI Content Generator**  
    - Type: LangChain OpenAI  
    - Role: Sends a detailed prompt with source content and metadata to GPT-5.1, requesting JSON output with specific structure including LinkedIn posts, Twitter threads, email newsletters, key takeaways, quotes, and video scripts  
    - Config: Model gpt-5.1, max tokens 4096, temperature 0.7  
    - Inputs: Extracted and normalized content JSON from blog, YouTube, or raw text nodes  
    - Outputs: AI-generated JSON content encapsulated in the message content  
    - Edge Cases: API rate limits, invalid JSON response, prompt failures, partial output, timeout.

---

#### 2.4 Document Formatting & Output

- **Overview:**  
  Parses AI JSON, formats a comprehensive Google Doc with sections for each content type, creates and updates the Google Doc, and prepares a structured response with document details.

- **Nodes Involved:**  
  - Format Document Content  
  - Create Google Doc  
  - Write Content  
  - Prepare Response  
  - Edit Fields

- **Node Details:**  

  - **Format Document Content**  
    - Type: Code  
    - Role: Parses AI JSON response safely, builds markdown-like content string for Google Docs with sections: Analysis, LinkedIn posts, Twitter thread, Email newsletter, Key takeaways, Quote cards, Video script, and metadata header  
    - Inputs: AI Content Generator output + source content extraction nodes for metadata  
    - Outputs: Object containing `docTitle`, `docContent`, source metadata, analysis, and timestamp  
    - Edge Cases: AI response parsing failure, missing sections in AI output, empty content, error handling if source content missing.

  - **Create Google Doc**  
    - Type: Google Docs node  
    - Role: Creates a new Google Doc with the formatted document title  
    - Inputs: Format Document Content output (docTitle)  
    - Credentials: Google Docs OAuth2  
    - Outputs: Document metadata including documentId  
    - Edge Cases: API quota, auth failure.

  - **Write Content**  
    - Type: Google Docs node  
    - Role: Updates the created Google Doc by inserting the formatted content text  
    - Inputs: Document ID from Create Google Doc, content from Format Document Content  
    - Credentials: Google Docs OAuth2  
    - Edge Cases: API failures, document locking, network issues.

  - **Prepare Response**  
    - Type: Code  
    - Role: Prepares the final JSON response to the user/webhook caller containing success flag, Google Doc URL, document ID, titles, source info, and analysis data  
    - Inputs: Write Content output + Format Document Content data  
    - Outputs: Final structured response JSON  
    - Edge Cases: Missing document ID or incomplete data.

  - **Edit Fields**  
    - Type: Set  
    - Role: Final formatting of response to include document URL under `documentUrl` property in output JSON  
    - Inputs: Prepare Response output  
    - Outputs: Final output for webhook response.

---

### 3. Summary Table

| Node Name                | Node Type                 | Functional Role                          | Input Node(s)                    | Output Node(s)                  | Sticky Note                                                                                                   |
|--------------------------|---------------------------|----------------------------------------|---------------------------------|---------------------------------|--------------------------------------------------------------------------------------------------------------|
| Sticky Note              | Sticky Note               | Overview and branding                   | None                            | None                            | ## 🎨 AI Content Repurposing Engine ... [Agentical AI](https://agenticalai.com)                               |
| Sticky Note1             | Sticky Note               | Content Input explanation               | None                            | None                            | ### 1️⃣ Content Input ... form accepts 3 input types                                                         |
| Sticky Note2             | Sticky Note               | Content Extraction explanation          | None                            | None                            | ### 2️⃣ Content Extraction ... unified format for AI                                                          |
| Sticky Note3             | Sticky Note               | AI Content Generation explanation       | None                            | None                            | ### 3️⃣ AI Content Generation ... platform-optimized content                                                 |
| Sticky Note4             | Sticky Note               | Google Docs Output explanation          | None                            | None                            | ### 4️⃣ Google Docs Output ... ready to copy/paste                                                           |
| Content Input Form       | Form Trigger              | Input collection via form                | None                            | Is Blog URL?                    |                                                                                                              |
| Is Blog URL?             | IF                        | Branch if input is Blog URL              | Content Input Form              | Fetch Blog Page / Is YouTube URL? |                                                                                                              |
| Is YouTube URL?          | IF                        | Branch if input is YouTube URL           | Is Blog URL?                   | Extract Video ID / Process Raw Text |                                                                                                              |
| Fetch Blog Page          | HTTP Request              | Fetch blog HTML                          | Is Blog URL?                   | Extract Blog Content            |                                                                                                              |
| Extract Blog Content     | Code                      | Parse blog HTML, extract title & content| Fetch Blog Page                | AI Content Generator            |                                                                                                              |
| Extract Video ID         | Code                      | Extract YouTube video ID                 | Is YouTube URL?                | Get YouTube Metadata            |                                                                                                              |
| Get YouTube Metadata     | HTTP Request              | Fetch YouTube video metadata via API    | Extract Video ID               | Fetch YouTube Page              |                                                                                                              |
| Fetch YouTube Page       | HTTP Request              | Fetch YouTube page HTML for captions    | Get YouTube Metadata           | Process YouTube Data            |                                                                                                              |
| Process YouTube Data     | Code                      | Aggregate metadata and captions         | Extract Video ID, Get YouTube Metadata, Fetch YouTube Page | AI Content Generator         |                                                                                                              |
| Process Raw Text         | Code                      | Pass raw text with minimal processing   | Is YouTube URL? false branch   | AI Content Generator            |                                                                                                              |
| AI Content Generator     | OpenAI (LangChain)        | Generate multi-format repurposed content| Extract Blog Content, Process YouTube Data, Process Raw Text | Format Document Content        |                                                                                                              |
| Format Document Content  | Code                      | Parse AI JSON, format Google Doc content| AI Content Generator           | Create Google Doc               |                                                                                                              |
| Create Google Doc        | Google Docs               | Create new Google Doc                    | Format Document Content        | Write Content                  |                                                                                                              |
| Write Content            | Google Docs               | Insert formatted content into Doc       | Create Google Doc              | Prepare Response               |                                                                                                              |
| Prepare Response        | Code                      | Prepare final response JSON              | Write Content                 | Edit Fields                   |                                                                                                              |
| Edit Fields              | Set                       | Format final output fields               | Prepare Response              | None                          |                                                                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Form Trigger node**  
   - Name: `Content Input Form`  
   - Path: `/content-repurpose`  
   - Add form fields:  
     - Dropdown `Content Type` with options: Blog Post URL, YouTube Video URL, Raw Text / Paste Content (required)  
     - Textarea `Content Input` (required)  
     - Text `Target Audience (Optional)`  
     - Textarea `Additional Instructions (Optional)`  
   - Add a form description explaining usage and credential requirements.

2. **Add an IF node `Is Blog URL?`**  
   - Condition: `$json["Content Type"] === "Blog Post URL"`  
   - Connect from `Content Input Form` output.

3. **Add an IF node `Is YouTube URL?`**  
   - Condition: `$json["Content Type"] === "YouTube Video URL"`  
   - Connect false branch from `Is Blog URL?`.

4. **For Blog input path:**  
   - Add HTTP Request `Fetch Blog Page`:  
     - URL: `={{ $('Content Input Form').item.json['Content Input'].trim() }}`  
     - Timeout: 30 seconds  
     - Expect response as text  
     - Connect true branch of `Is Blog URL?` to this node.

   - Add Code node `Extract Blog Content`:  
     - JavaScript to parse HTML, extract title, description, author, clean main content, truncate if needed (see logic in node)  
     - Input: `Fetch Blog Page` + form data for metadata fields  
     - Connect output of `Fetch Blog Page`.

5. **For YouTube input path:**  
   - Add Code node `Extract Video ID`:  
     - JavaScript to parse various YouTube URL formats and extract video ID  
     - Input: from `Is YouTube URL?` true branch

   - Add HTTP Request `Get YouTube Metadata`:  
     - URL: `https://www.googleapis.com/youtube/v3/videos`  
     - Query parameters: `part=snippet,contentDetails,statistics`, `id={{ $json.videoId }}`  
     - Use YouTube OAuth2 credential  
     - Connect from `Extract Video ID`

   - Add HTTP Request `Fetch YouTube Page`:  
     - URL: `https://www.youtube.com/watch?v={{ $('Extract Video ID').item.json.videoId }}`  
     - Timeout: 15 seconds  
     - Connect from `Get YouTube Metadata`

   - Add Code node `Process YouTube Data`:  
     - JavaScript to combine metadata and scraped HTML for captions, build markdown content, truncate if needed  
     - Connect outputs from `Extract Video ID`, `Get YouTube Metadata`, and `Fetch YouTube Page`.

6. **For Raw Text input path:**  
   - Add Code node `Process Raw Text`:  
     - JavaScript to pass raw text through, attempt to extract title from first line, truncate if needed  
     - Connect false branch of `Is YouTube URL?`.

7. **AI Content Generation:**  
   - Add LangChain OpenAI node `AI Content Generator`:  
     - Model: gpt-5.1  
     - Temperature: 0.7  
     - Max tokens: 4096  
     - Prompt: Detailed prompt instructing GPT to analyze content and produce JSON with multiple content formats (LinkedIn, Twitter, Email, Quotes, Video Script)  
     - Input: Connect outputs of `Extract Blog Content`, `Process YouTube Data`, and `Process Raw Text` (all routed to this node)

   - Configure OpenAI credentials.

8. **Document Formatting:**  
   - Add Code node `Format Document Content`:  
     - Parses AI JSON output, builds markdown-like content string organized by content types and metadata  
     - Connect from `AI Content Generator`.

9. **Google Docs Output:**  
   - Add Google Docs node `Create Google Doc`:  
     - Title: `={{ $json.docTitle }}`  
     - Folder ID: default or specific folder  
     - Connect from `Format Document Content`.

   - Add Google Docs node `Write Content`:  
     - Operation: update document  
     - Document URL/ID: use newly created document ID  
     - Actions: insert text from `Format Document Content` (`docContent`)  
     - Connect from `Create Google Doc`.

   - Configure Google Docs OAuth2 credentials.

10. **Prepare Final Response:**  
    - Add Code node `Prepare Response`:  
      - Build JSON response with success, document URL, ID, title, source info, analysis, timestamp  
      - Connect from `Write Content`.

11. **Edit Fields (Set):**  
    - Add Set node `Edit Fields`:  
      - Assign `documentUrl` property for output  
      - Connect from `Prepare Response`.

12. **Connect workflows:**  
    - From `Content Input Form` → `Is Blog URL?`  
    - True → Blog path nodes → `AI Content Generator`  
    - False → `Is YouTube URL?`  
    - True → YouTube path nodes → `AI Content Generator`  
    - False → Raw text node → `AI Content Generator`  
    - Then `AI Content Generator` → `Format Document Content` → Google Docs nodes → Final response nodes.

---

### 5. General Notes & Resources

| Note Content                                                                                             | Context or Link                          |
|--------------------------------------------------------------------------------------------------------|----------------------------------------|
| The workflow uses OpenAI GPT-5.1 (GPT-4o) for advanced content generation with structured JSON output. | Requires OpenAI API credential          |
| Google Docs integration outputs a fully formatted doc ready for content repurposing.                    | Requires Google Docs OAuth2 credentials |
| Optional YouTube API improves video metadata accuracy but is not mandatory.                             | YouTube OAuth2 API credential optional |
| Longer input content produces better AI outputs; blog posts are ideal, YouTube videos need captions enabled. | Tips from workflow sticky note          |
| Created by Agentical AI — https://agenticalai.com                                                      | Branding and creator link                |

---

**Disclaimer:** The provided text is exclusively from an automated n8n workflow. It adheres strictly to content policies, containing no illegal or offensive material. All data handled is legal and publicly available.