🔍🛠️Generate SEO-Optimized WordPress Content with AI Powered Perplexity Research

https://n8nworkflows.xyz/workflows/-----generate-seo-optimized-wordpress-content-with-ai-powered-perplexity-research-3291


# 🔍🛠️Generate SEO-Optimized WordPress Content with AI Powered Perplexity Research

### 1. Workflow Overview

This workflow automates the creation and publishing of SEO-optimized blog posts on WordPress by leveraging AI-powered research and content generation. It is designed for content creators, marketers, and businesses, especially those in AI consulting and workflow automation, who want to streamline blog production with minimal manual effort.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Captures user input via a form submission containing the research query.
- **1.2 Perplexity AI Research:** Performs detailed topic research using the Perplexity AI API.
- **1.3 Research Cleanup:** Processes and formats the raw research output for downstream use.
- **1.4 AI Content Generation:** Uses OpenAI models to generate a comprehensive blog post draft based on the research.
- **1.5 SEO Metadata Creation:** Creates SEO-friendly title, slug, and meta description using AI.
- **1.6 HTML Content Creation:** Converts the blog draft into WordPress-compatible HTML format.
- **1.7 Content Aggregation:** Combines generated content and metadata for publishing.
- **1.8 WordPress Publishing:** Uploads the blog post as a draft to WordPress.
- **1.9 Image Handling:** Downloads a featured image, uploads it to WordPress media, and sets it as the post’s featured image.
- **1.10 Notification:** Sends a Telegram message confirming successful blog post creation.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:** Captures the user’s research query via a web form to initiate the workflow.
- **Nodes Involved:**  
  - On form submission

- **Node Details:**

  - **On form submission**  
    - Type: Form Trigger  
    - Role: Entry point capturing user input (research query) from a form titled "Blog Factory".  
    - Configuration: Single textarea field labeled "Research Query" with a placeholder example and required validation.  
    - Inputs: None (trigger node)  
    - Outputs: Emits JSON containing the research query.  
    - Edge Cases: Missing or empty query input will block execution; ensure form validation is enabled.  
    - Version: 2.2  

---

#### 2.2 Perplexity AI Research

- **Overview:** Sends the user query to Perplexity AI API to retrieve detailed research summaries from reputable sources.
- **Nodes Involved:**  
  - Perplexity Research  
  - Cleanup Links

- **Node Details:**

  - **Perplexity Research**  
    - Type: HTTP Request  
    - Role: Calls Perplexity AI chat completions endpoint with a system prompt to act as a professional news researcher and user prompt containing the research query.  
    - Configuration: POST request with JSON body including model "sonar-pro" and messages array. Uses HTTP header authentication with stored credentials.  
    - Inputs: Research query from form submission node.  
    - Outputs: Raw research content with citations.  
    - Edge Cases: API authentication failure, rate limits, network timeouts, malformed query input.  
    - Version: 4.2  

  - **Cleanup Links**  
    - Type: Set  
    - Role: Cleans up the research text by replacing numeric citation placeholders ([1], [2], etc.) with corresponding source URLs from the API response citations array.  
    - Configuration: Uses JavaScript expressions with replaceAll to substitute citation markers with " - source: [URL]" strings.  
    - Inputs: Raw research output from Perplexity Research node.  
    - Outputs: Cleaned research text with embedded source references.  
    - Edge Cases: Missing or fewer citations than placeholders may cause undefined replacements; ensure citations array length matches placeholders.  
    - Version: 3.4  

---

#### 2.3 AI Content Generation

- **Overview:** Generates a detailed, SEO-optimized blog post draft using OpenAI based on the cleaned research and query.
- **Nodes Involved:**  
  - Copywriter AI Agent

- **Node Details:**

  - **Copywriter AI Agent**  
    - Type: Langchain Agent (OpenAI)  
    - Role: Produces a comprehensive blog post draft (1500-2000 words) tailored for AI consulting and workflow automation industry in Canada, incorporating the research and query.  
    - Configuration: Prompt includes instructions for tone, length, keyword integration, structure, and call-to-action. Inputs are the research query and cleaned research findings.  
    - Inputs: Cleaned research text from Cleanup Links node, original query from form submission.  
    - Outputs: Full blog post draft text.  
    - Edge Cases: API errors, incomplete generation, prompt misinterpretation, token limits.  
    - Version: 1.8  

---

#### 2.4 SEO Metadata Creation

- **Overview:** Creates SEO-friendly slug, title, and meta description for the blog post using AI.
- **Nodes Involved:**  
  - Create Title, Slug, Meta  
  - Structured Output Parser  
  - gpt-4o-mini (supporting OpenAI model)

- **Node Details:**

  - **Create Title, Slug, Meta**  
    - Type: Langchain Agent  
    - Role: Generates a JSON object with slug, title, and meta description following strict SEO guidelines, avoiding overused AI buzzwords and formatting issues.  
    - Configuration: Prompt includes detailed rules for slug conciseness, title engagement, and meta description length and keyword usage.  
    - Inputs: Blog post draft text from Copywriter AI Agent and output from Structured Output Parser.  
    - Outputs: JSON with slug, title, meta description.  
    - Edge Cases: Parsing errors if output is not valid JSON, incomplete or off-topic metadata.  
    - Version: 1.8  

  - **Structured Output Parser**  
    - Type: Langchain Structured Output Parser  
    - Role: Parses AI output into structured JSON to feed into Create Title, Slug, Meta node.  
    - Configuration: Uses a JSON schema example with slug, title, and meta fields.  
    - Inputs: Output from gpt-4o-mini node.  
    - Outputs: Parsed JSON for metadata generation.  
    - Edge Cases: Schema mismatch, parsing failures.  
    - Version: 1.2  

  - **gpt-4o-mini**  
    - Type: Langchain OpenAI Chat Model  
    - Role: Supports metadata generation by providing AI completions.  
    - Configuration: Uses GPT-4o-mini model.  
    - Inputs: Blog post draft text.  
    - Outputs: Raw AI completions for metadata.  
    - Edge Cases: API errors, token limits.  
    - Version: 1.2  

---

#### 2.5 HTML Content Creation

- **Overview:** Converts the blog post draft into WordPress-compatible HTML with specific styling and structure.
- **Nodes Involved:**  
  - Create HTML  
  - Merge  
  - Cleanup HTML

- **Node Details:**

  - **Create HTML**  
    - Type: Langchain OpenAI  
    - Role: Generates raw HTML content formatted for WordPress blocks, including title, reading time, key takeaways, TOC, main content, and FAQ, with styling and hyperlink rules.  
    - Configuration: Prompt enforces no emojis, specific color styling (#00c2ff), heading IDs, and WordPress block classes.  
    - Inputs: Blog post draft text from Copywriter AI Agent.  
    - Outputs: Raw HTML content string.  
    - Edge Cases: HTML formatting errors, incomplete output, token limits.  
    - Version: 1.8  

  - **Merge**  
    - Type: Merge  
    - Role: Aggregates outputs from Create HTML, Create Title, Slug, Meta, and Copywriter AI Agent for combined processing.  
    - Configuration: Merges three inputs.  
    - Inputs: Outputs from Create HTML, Create Title, Slug, Meta, and Copywriter AI Agent.  
    - Outputs: Combined data object.  
    - Edge Cases: Data mismatch or missing inputs.  
    - Version: 3  

  - **Cleanup HTML**  
    - Type: Set  
    - Role: Cleans the HTML content by removing markdown code fences (```html and ```) from the generated HTML string.  
    - Configuration: Uses string replaceAll expressions.  
    - Inputs: Merged data from previous node.  
    - Outputs: Clean HTML content ready for WordPress.  
    - Edge Cases: Unexpected formatting in HTML string.  
    - Version: 3.4  

---

#### 2.6 WordPress Publishing

- **Overview:** Publishes the blog post draft with metadata and content to WordPress.
- **Nodes Involved:**  
  - Wordpress

- **Node Details:**

  - **Wordpress**  
    - Type: WordPress node  
    - Role: Creates a new WordPress post in draft status with title, slug, content, author ID, and closed comments.  
    - Configuration: Title, slug, and content are dynamically set from combined AI outputs. Author ID is fixed at 2. Post status is draft, comments disabled.  
    - Inputs: Cleaned HTML content and metadata from Cleanup HTML node.  
    - Outputs: WordPress post creation response including post ID.  
    - Edge Cases: API authentication failure, invalid content, slug conflicts, network issues.  
    - Version: 1  

---

#### 2.7 Image Handling

- **Overview:** Downloads a featured image from a fixed URL, uploads it to WordPress media library, and sets it as the featured image for the post.
- **Nodes Involved:**  
  - Set Image URL  
  - GET Image  
  - Upload Image to Wordpress  
  - Set Image on Wordpress Post

- **Node Details:**

  - **Set Image URL**  
    - Type: Set  
    - Role: Defines a static image URL to be used as the featured image.  
    - Configuration: Hardcoded URL to an image hosted externally.  
    - Inputs: None (triggered after WordPress post creation).  
    - Outputs: JSON with "image-url" field.  
    - Edge Cases: Static URL may become invalid; consider dynamic image selection for robustness.  
    - Version: 3.4  

  - **GET Image**  
    - Type: HTTP Request  
    - Role: Downloads the image binary data from the URL set in the previous node.  
    - Configuration: GET request to the image URL.  
    - Inputs: Image URL from Set Image URL node.  
    - Outputs: Binary image data.  
    - Edge Cases: Image URL unreachable, network timeouts, large file size.  
    - Version: 4.2  

  - **Upload Image to Wordpress**  
    - Type: HTTP Request  
    - Role: Uploads the downloaded image binary to WordPress media endpoint.  
    - Configuration: POST request to WordPress media API with binary data, sets Content-Disposition header with filename referencing the WordPress post ID. Uses WordPress API credentials.  
    - Inputs: Binary image data from GET Image node, WordPress post ID from Wordpress node.  
    - Outputs: Media upload response including media ID.  
    - Edge Cases: Authentication failure, file size limits, API errors.  
    - Version: 4.2  

  - **Set Image on Wordpress Post**  
    - Type: HTTP Request  
    - Role: Updates the WordPress post to set the uploaded media as the featured image using the media ID.  
    - Configuration: POST request to WordPress posts endpoint with query parameter "featured_media" set to media ID. Uses WordPress API credentials.  
    - Inputs: Media ID from Upload Image to Wordpress node, WordPress post ID from Wordpress node.  
    - Outputs: Post update confirmation.  
    - Edge Cases: API errors, invalid media ID, permission issues.  
    - Version: 4.2  

---

#### 2.8 Notification

- **Overview:** Sends a Telegram message to notify that the blog post creation process completed successfully.
- **Nodes Involved:**  
  - Send Success Message to Telegram

- **Node Details:**

  - **Send Success Message to Telegram**  
    - Type: Telegram node  
    - Role: Sends a text message to a configured Telegram chat ID confirming success with timestamp.  
    - Configuration: Uses environment variable for chat ID, disables attribution append.  
    - Inputs: Triggered after image is set on WordPress post.  
    - Outputs: Telegram API response.  
    - Edge Cases: Invalid chat ID, bot permissions, network issues.  
    - Version: 1.2  

---

### 3. Summary Table

| Node Name                  | Node Type                               | Functional Role                          | Input Node(s)                  | Output Node(s)                  | Sticky Note                                                                                   |
|----------------------------|---------------------------------------|----------------------------------------|-------------------------------|-------------------------------|----------------------------------------------------------------------------------------------|
| On form submission         | Form Trigger                          | Captures user research query            | None                          | Perplexity Research            |                                                                                              |
| Perplexity Research        | HTTP Request                         | Performs AI research on query           | On form submission            | Cleanup Links                  | ## Perplexity Research                                                                       |
| Cleanup Links              | Set                                  | Cleans research text with source links | Perplexity Research           | Copywriter AI Agent            |                                                                                              |
| Copywriter AI Agent        | Langchain Agent                      | Generates full blog post draft           | Cleanup Links                 | Create HTML, Create Title, Slug, Meta, Merge |                                                                                              |
| Create HTML                | Langchain OpenAI                     | Generates WordPress-compatible HTML     | Copywriter AI Agent           | Merge                         | ## Create HTML                                                                              |
| Create Title, Slug, Meta   | Langchain Agent                      | Creates SEO metadata JSON                | Copywriter AI Agent, Structured Output Parser, gpt-4o-mini | Merge                         | ## Create Title, Slug & Meta                                                                |
| Structured Output Parser   | Langchain Output Parser              | Parses AI metadata output to JSON       | gpt-4o-mini                   | Create Title, Slug, Meta       |                                                                                              |
| gpt-4o-mini                | Langchain OpenAI Chat Model          | Supports metadata generation             | Copywriter AI Agent           | Structured Output Parser       |                                                                                              |
| Merge                     | Merge                                | Aggregates content and metadata          | Create HTML, Create Title, Slug, Meta, Copywriter AI Agent | Combine Blog Details           |                                                                                              |
| Combine Blog Details       | Aggregate                           | Aggregates all blog data                  | Merge                        | Cleanup HTML                  |                                                                                              |
| Cleanup HTML               | Set                                  | Cleans HTML content                      | Combine Blog Details          | Wordpress                     |                                                                                              |
| Wordpress                 | WordPress                            | Creates draft blog post                   | Cleanup HTML                 | Set Image URL                 | ## Post on Wordpress                                                                        |
| Set Image URL             | Set                                  | Defines static featured image URL        | Wordpress                    | GET Image                    | ## Set Image for Wordpress Post                                                             |
| GET Image                 | HTTP Request                         | Downloads image binary                    | Set Image URL                | Upload Image to Wordpress     |                                                                                              |
| Upload Image to Wordpress | HTTP Request                         | Uploads image to WordPress media         | GET Image                   | Set Image on Wordpress Post   |                                                                                              |
| Set Image on Wordpress Post| HTTP Request                         | Sets uploaded image as featured image   | Upload Image to Wordpress    | Send Success Message to Telegram |                                                                                              |
| Send Success Message to Telegram | Telegram                        | Sends success notification               | Set Image on Wordpress Post  | None                         |                                                                                              |
| Sticky Note4              | Sticky Note                         | Notes on SEO optimized blog post         | None                        | None                         | ## Write SEO Optimized Blog Post                                                            |
| Sticky Note5              | Sticky Note                         | Notes on Perplexity Research              | None                        | None                         | ## Perplexity Research                                                                       |
| Sticky Note6              | Sticky Note                         | Notes on HTML creation                     | None                        | None                         | ## Create HTML                                                                              |
| Sticky Note7              | Sticky Note                         | Notes on Title, Slug & Meta creation      | None                        | None                         | ## Create Title, Slug & Meta                                                                |
| Sticky Note8              | Sticky Note                         | Notes on WordPress posting                 | None                        | None                         | ## Post on Wordpress                                                                        |
| Sticky Note1              | Sticky Note                         | Notes on setting image for WordPress post | None                        | None                         | ## Set Image for Wordpress Post                                                             |
| Sticky Note               | Sticky Note                         | Sample generic search terms for research  | None                        | None                         | Sample Generic Search Terms: 10 example queries for geo-specific research                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Form Trigger Node ("On form submission")**  
   - Type: Form Trigger  
   - Configure form title: "Blog Factory"  
   - Add one textarea field: Label "Research Query", placeholder with example query, required.  
   - Save webhook ID for external form submission.

2. **Add HTTP Request Node ("Perplexity Research")**  
   - Method: POST  
   - URL: https://api.perplexity.ai/chat/completions  
   - Authentication: HTTP Header Auth with Perplexity API credentials  
   - Body (JSON):  
     ```json
     {
       "model": "sonar-pro",
       "messages": [
         {"role": "system", "content": "Act as a professional news researcher who is capable of finding detailed summaries about a news topic from highly reputable sources."},
         {"role": "user", "content": "Research the following topic and return everything you can find about: '{{ $json[\"Research Query\"] }}'."}
       ]
     }
     ```  
   - Connect input from Form Trigger node.

3. **Add Set Node ("Cleanup Links")**  
   - Use JavaScript expressions to replace citation placeholders [1], [2], etc. with corresponding URLs from the citations array in the Perplexity response.  
   - Assign cleaned text to a new field "research".

4. **Add Langchain Agent Node ("Copywriter AI Agent")**  
   - Model: GPT-4o-mini or equivalent  
   - Prompt: Detailed instructions to write a 1500-2000 word SEO-optimized blog post for AI consulting and workflow automation industry in Canada, incorporating research and query.  
   - Inputs: research text from Cleanup Links, original query from Form Trigger.

5. **Add Langchain OpenAI Node ("Create HTML")**  
   - Model: GPT-4o-mini  
   - Prompt: Generate WordPress-compatible HTML with specified structure and styling, based on blog post draft from Copywriter AI Agent.

6. **Add Langchain Agent Node ("Create Title, Slug, Meta")**  
   - Prompt: Generate JSON with slug, title, and meta description following SEO guidelines, based on blog post draft.

7. **Add Langchain Structured Output Parser Node ("Structured Output Parser")**  
   - JSON schema example with slug, title, meta fields.  
   - Connect output from gpt-4o-mini node supporting metadata generation.

8. **Add Langchain OpenAI Chat Model Node ("gpt-4o-mini")**  
   - Model: GPT-4o-mini  
   - Connect input from Copywriter AI Agent to support metadata generation.

9. **Add Merge Node ("Merge")**  
   - Configure to merge three inputs: Create HTML, Create Title, Slug, Meta, and Copywriter AI Agent outputs.

10. **Add Aggregate Node ("Combine Blog Details")**  
    - Aggregate all merged data into a single item.

11. **Add Set Node ("Cleanup HTML")**  
    - Remove markdown code fences from HTML content field.

12. **Add WordPress Node ("Wordpress")**  
    - Credentials: WordPress API credentials  
    - Action: Create post  
    - Parameters:  
      - Title: From combined metadata (slug/title)  
      - Slug: From metadata  
      - Content: Cleaned HTML content  
      - Status: Draft  
      - Author ID: 2 (or your author ID)  
      - Comment status: Closed

13. **Add Set Node ("Set Image URL")**  
    - Assign a static image URL string to "image-url" field.

14. **Add HTTP Request Node ("GET Image")**  
    - Method: GET  
    - URL: From "image-url" field  
    - Output: Binary data

15. **Add HTTP Request Node ("Upload Image to Wordpress")**  
    - Method: POST  
    - URL: https://yourwordpresssite.com/wp-json/wp/v2/media  
    - Authentication: WordPress API credentials  
    - Headers: Content-Disposition with filename referencing WordPress post ID  
    - Body: Binary data from GET Image node

16. **Add HTTP Request Node ("Set Image on Wordpress Post")**  
    - Method: POST  
    - URL: https://yourwordpresssite.com/wp-json/wp/v2/posts/{{post_id}}  
    - Query parameter: featured_media = media ID from upload response  
    - Authentication: WordPress API credentials

17. **Add Telegram Node ("Send Success Message to Telegram")**  
    - Credentials: Telegram Bot API credentials  
    - Chat ID: Use environment variable or fixed chat ID  
    - Message: "Success! Your blog post was created at {{ $now }}"

18. **Connect nodes in the following order:**  
    - On form submission → Perplexity Research → Cleanup Links → Copywriter AI Agent →  
      → Create HTML → Merge  
      → Create Title, Slug, Meta → Merge  
      → Combine Blog Details → Cleanup HTML → Wordpress →  
      → Set Image URL → GET Image → Upload Image to Wordpress → Set Image on Wordpress Post →  
      → Send Success Message to Telegram

19. **Credential Setup:**  
    - Configure WordPress API credentials with appropriate permissions for post creation and media upload.  
    - Configure OpenAI API credentials.  
    - Configure Perplexity AI API credentials (HTTP header auth).  
    - Configure Telegram Bot credentials with chat ID.

20. **Testing:**  
    - Test with sample queries to verify research retrieval, content generation, WordPress post creation, image upload, and Telegram notification.

---

### 5. General Notes & Resources

| Note Content                                                                                                   | Context or Link                                                                                                   |
|---------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------|
| This workflow is ideal for AI consulting and workflow automation professionals looking to automate blog creation.| Workflow description and target audience.                                                                        |
| Sample generic search terms are provided as inspiration for research queries, including sector-specific challenges.| Sticky Note with 10 example search terms for geographic or industry customization.                               |
| The workflow uses GPT-4o-mini model for content generation and metadata creation, balancing quality and cost.  | Model selection note.                                                                                            |
| WordPress post is created as draft with author ID 2 and comments disabled to allow review before publishing.  | Publishing configuration note.                                                                                   |
| Image URL is statically set but can be customized or made dynamic for better content relevance.                | Image handling customization suggestion.                                                                         |
| Telegram notifications provide real-time success alerts to keep users informed of workflow completion.        | Notification feature for operational monitoring.                                                                 |
| For detailed WordPress API documentation, see: https://developer.wordpress.org/rest-api/reference/posts/      | External resource for WordPress API usage.                                                                        |
| For OpenAI API usage and prompt design, see: https://platform.openai.com/docs/guides/chat                      | External resource for OpenAI integration.                                                                         |
| Perplexity AI API documentation is required for authentication and request format details.                     | External resource for Perplexity AI integration (credentials setup).                                              |

---

This documentation provides a complete, structured reference to understand, reproduce, and modify the workflow for generating SEO-optimized WordPress content using AI-powered Perplexity research.