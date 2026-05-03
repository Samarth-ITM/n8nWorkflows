Generate Multiple Languange Blogpost with OpenAI, Support Yoast & Polylang

https://n8nworkflows.xyz/workflows/generate-multiple-languange-blogpost-with-openai--support-yoast---polylang-5287


# Generate Multiple Languange Blogpost with OpenAI, Support Yoast & Polylang

---

## 1. Workflow Overview

This workflow automates the generation of multilingual blog posts using OpenAI’s language models, integrates with WordPress, and supports SEO metadata management with Yoast as well as multilingual content handling through Polylang. Its target use case is content marketing teams or agencies managing visa and education consultation blogs that require dynamic, SEO-friendly, and localized content generation and publishing.

The workflow is logically divided into two primary blocks:

- **1.1 Polylang Debugging and Analysis Block**: This block fetches recent posts, taxonomies, and API routes from WordPress to analyze how Polylang manages multilingual content and language metadata via the WordPress REST API. It includes diagnostic code nodes to detect language-related fields, taxonomies, and API endpoints, providing insights for proper multilingual integration.

- **1.2 Article Generation and Publishing Block**: This block is triggered by a scheduled event to generate a new blog post topic and metadata using OpenAI, then generate the article content, create a WordPress draft post, set Yoast SEO meta fields, generate and upload a featured image, and finally attach the image to the post. It leverages OpenAI for natural language generation and WordPress REST API for content management.

---

## 2. Block-by-Block Analysis

### 2.1 Polylang Debugging and Analysis Block

**Overview:**  
This block collects data from the WordPress REST API related to posts, taxonomies, and API routes. It then runs JavaScript code nodes to analyze how Polylang and multilingual data are represented in the API responses, identifying fields, taxonomies, and endpoints relevant for language handling. This aids in understanding Polylang’s data structure and preparing for seamless multilingual content creation.

**Nodes Involved:**  
- Manual Trigger  
- Get Recent Posts  
- Get Single Post Details  
- Get All Taxonomies  
- Analyze Taxonomies (Code)  
- Get Posts Schema  
- Check PLL Endpoint  
- Get API Routes  
- Analyze API Routes (Code)  
- Analyze Post Structure (Code)  
- Get Terms  
- Sticky Note (Polylang Debugging)

**Node Details:**

- **Manual Trigger**  
  - *Type:* Trigger node for manual execution.  
  - *Role:* Starts the debugging workflow manually.  
  - *Config:* No parameters.  
  - *Connections:* Outputs to Get Recent Posts, Get All Taxonomies, Get Posts Schema, Get API Routes.  
  - *Failures:* None expected; manual start.  

- **Get Recent Posts**  
  - *Type:* HTTP Request (WordPress REST API).  
  - *Role:* Fetches the 5 most recent posts from `/wp-json/wp/v2/posts`.  
  - *Config:* Uses predefined WordPress credentials. Full HTTP response enabled for debugging.  
  - *Output:* JSON array of posts metadata.  
  - *Failures:* Auth errors, connection issues, malformed response.  

- **Get Single Post Details**  
  - *Type:* HTTP Request.  
  - *Role:* Fetches detailed data for a single post (dynamically using post ID).  
  - *Config:* URL templated with `{{ $json.id }}` from previous node. `_embed` parameter to include linked resources.  
  - *Output:* Detailed post JSON with embedded data.  
  - *Failures:* Missing post ID, HTTP errors.  

- **Get All Taxonomies**  
  - *Type:* HTTP Request.  
  - *Role:* Retrieves taxonomy definitions from `/wp-json/wp/v2/taxonomies`.  
  - *Config:* Uses WordPress credentials, no special parameters.  
  - *Output:* JSON object of taxonomies.  
  - *Failures:* Auth or network errors.  

- **Analyze Taxonomies (Code Node)**  
  - *Type:* Code (JavaScript).  
  - *Role:* Parses taxonomy data to identify language-related taxonomies (e.g., containing "language" or "translation").  
  - *Config:* Analyzes keys and properties, builds recommendation array.  
  - *Input:* Taxonomy JSON from previous node.  
  - *Output:* Structured findings including language taxonomies and suggestions.  
  - *Edge Cases:* Empty input, unexpected taxonomy structure.  

- **Get Posts Schema**  
  - *Type:* HTTP Request.  
  - *Role:* Calls OPTIONS on `/wp-json/wp/v2/posts` to retrieve schema and supported methods, helping understand fields available.  
  - *Config:* Full HTTP response enabled.  
  - *Output:* Schema info for posts endpoint.  
  - *Failure:* HTTP method not allowed or API changes.  

- **Check PLL Endpoint**  
  - *Type:* HTTP Request.  
  - *Role:* Queries `/wp-json/pll/v1` to check if Polylang REST API endpoints are available.  
  - *Config:* Set to never error on response (returns empty or error JSON instead).  
  - *Output:* Polylang endpoint presence and data if available.  
  - *Failures:* Endpoint missing, plugin deactivated.  

- **Get API Routes**  
  - *Type:* HTTP Request.  
  - *Role:* Retrieves main API root `/wp-json` to discover namespaces and routes for language or Polylang integration.  
  - *Config:* Standard GET, authenticated.  
  - *Output:* JSON describing API namespaces and routes.  
  - *Failures:* Network errors, permission denied.  

- **Analyze API Routes (Code Node)**  
  - *Type:* Code (JavaScript).  
  - *Role:* Parses API root data to detect Polylang or language-related namespaces and routes.  
  - *Input:* API routes JSON.  
  - *Output:* List of detected Polylang namespaces/routes and recommendations.  
  - *Edge Cases:* Missing namespaces, unexpected API changes.  

- **Analyze Post Structure (Code Node)**  
  - *Type:* Code (JavaScript).  
  - *Role:* Performs deep analysis of a single post JSON to find language-related fields, meta, links, and embedded data to infer Polylang integration points.  
  - *Input:* Detailed post JSON from Get Single Post Details.  
  - *Output:* Collected fields with language hints, recommendations for multilingual handling.  
  - *Failures:* Empty input, unexpected data structures.  

- **Get Terms**  
  - *Type:* HTTP Request.  
  - *Role:* Fetches terms from `/wp-json/wp/v2/terms` with never error response to inspect taxonomy terms possibly related to language.  
  - *Failures:* API errors or empty term sets.  

- **Sticky Note (Polylang Debugging)**  
  - *Role:* Annotates the block with a header "Polylang Debugging" for clarity.  

---

### 2.2 Article Generation and Publishing Block

**Overview:**  
This block is scheduled weekly to automatically generate a blog post topic, metadata, and content using OpenAI’s GPT-4.1-mini model, create a WordPress draft post, enrich SEO metadata via Yoast fields, generate a fitting featured image with AI, upload it, and attach it to the post. It ensures multilingual content diversity and SEO best practices.

**Nodes Involved:**  
- Weekly Schedule  
- Generate Topic & Metadata (Chain LLM)  
- OpenAI Model - Topic1 (LM Chat OpenAI)  
- Parse Topic JSON1 (Output Parser Structured)  
- Generate Article Content (Agent LangChain)  
- OpenAI Model - Article1 (LM Chat OpenAI)  
- Create WordPress Draft  
- Set Yoast SEO Data  
- Generate Featured Image (OpenAI Image Generation)  
- Upload Image to WordPress  
- Attach Featured Image  
- Sticky Note1 (Article Generator)

**Node Details:**

- **Weekly Schedule**  
  - *Type:* Schedule Trigger.  
  - *Role:* Triggers the workflow every Wednesday at 10:00 AM weekly.  
  - *Config:* Interval-based weekly trigger, day=Wednesday, hour=10.  
  - *Failures:* None expected; time-based trigger.  

- **Generate Topic & Metadata (Chain LLM)**  
  - *Type:* LangChain Chain LLM.  
  - *Role:* Generates one blog post topic, language, and metadata formatted as JSON based on a defined prompt template.  
  - *Config:* Prompt instructs to pick a category (from WordPress categories), language (en, tr, fa), and generate SEO-friendly title, slug, meta description, and focus keyphrase. Output is structured JSON.  
  - *Input:* Triggered by schedule.  
  - *Output:* JSON topic metadata.  
  - *Failures:* Model timeout, malformed output, prompt issues.  

- **OpenAI Model - Topic1**  
  - *Type:* Chat OpenAI (GPT-4.1-mini).  
  - *Role:* Underlying model node invoked by Generate Topic & Metadata.  
  - *Config:* Temperature 0.7 for creativity.  
  - *Failures:* API quota, authentication, or response errors.  

- **Parse Topic JSON1**  
  - *Type:* Output Parser Structured.  
  - *Role:* Parses the JSON string output from the language model into usable JSON object with fields like category, language, title, slug, etc.  
  - *Failures:* Parsing errors if output is malformed.  

- **Generate Article Content (Agent LangChain)**  
  - *Type:* LangChain Agent.  
  - *Role:* Generates a full article (1000-1500 words) in HTML format without <h1>, following detailed structural and stylistic instructions for clarity, SEO, and practical guidance.  
  - *Config:* Uses prompt variables from generated topic metadata.  
  - *Failures:* Model errors, generation timeout, output formatting issues.  

- **OpenAI Model - Article1**  
  - *Type:* Chat OpenAI (GPT-4.1-mini).  
  - *Role:* Language model node used by Generate Article Content.  
  - *Config:* Max tokens 2500, temperature 0.8 for varied content.  
  - *Failures:* API errors or content generation failures.  

- **Create WordPress Draft**  
  - *Type:* WordPress node (REST API).  
  - *Role:* Creates a new draft post in WordPress with the generated title, slug, content (article HTML), author ID, and category.  
  - *Config:* Status set to draft, format standard, author fixed as ID 2.  
  - *Failures:* Auth errors, content validation, slug conflicts.  

- **Set Yoast SEO Data**  
  - *Type:* HTTP Request (WordPress REST API).  
  - *Role:* Updates the newly created post with Yoast SEO meta fields: focus keyword, meta description, and SEO title.  
  - *Config:* POST request to post endpoint with JSON body containing `meta` keys for Yoast fields.  
  - *Failures:* API errors, missing meta fields support, auth errors.  

- **Generate Featured Image (OpenAI Image Generation)**  
  - *Type:* OpenAI Image generation node.  
  - *Role:* Creates a cinematic and realistic featured image based on the blog post title with detailed visual style requirements for professionalism and relevance.  
  - *Config:* Size 1792x1024, style natural, quality standard. No text/flags/logos allowed.  
  - *Failures:* API quota, image generation failures.  

- **Upload Image to WordPress**  
  - *Type:* HTTP Request (WordPress REST API media endpoint).  
  - *Role:* Uploads the generated image as media in WordPress, attaching proper headers including Content-Disposition with filename based on post slug.  
  - *Config:* Sends binary image data, content type image/png.  
  - *Failures:* Upload errors, file size limits, auth issues.  

- **Attach Featured Image**  
  - *Type:* HTTP Request (WordPress REST API).  
  - *Role:* Updates the post to set the featured media ID with the uploaded image’s media ID.  
  - *Config:* POST request to update post, body parameter `featured_media` with image ID.  
  - *Failures:* Missing media ID, post update errors.  

- **Sticky Note1 (Article Generator)**  
  - *Role:* Annotates this block with "Article Generator" heading.  

---

## 3. Summary Table

| Node Name               | Node Type                         | Functional Role                                  | Input Node(s)                 | Output Node(s)                    | Sticky Note                     |
|-------------------------|----------------------------------|-------------------------------------------------|------------------------------|----------------------------------|--------------------------------|
| Manual Trigger          | manualTrigger                    | Start debugging and analysis block               | None                         | Get Recent Posts, Get All Taxonomies, Get Posts Schema, Get API Routes | Polylang Debugging             |
| Get Recent Posts        | httpRequest (WordPress API)       | Fetch recent posts                                | Manual Trigger               | Get Single Post Details           | Polylang Debugging             |
| Get Single Post Details | httpRequest (WordPress API)       | Fetch detailed post info                           | Get Recent Posts             | Analyze Post Structure            | Polylang Debugging             |
| Get All Taxonomies      | httpRequest (WordPress API)       | Fetch taxonomies                                  | Manual Trigger               | Analyze Taxonomies                | Polylang Debugging             |
| Analyze Taxonomies      | code                             | Analyze taxonomies for language fields            | Get All Taxonomies           | None                            | Polylang Debugging             |
| Get Posts Schema        | httpRequest (WordPress API)       | Retrieve post schema                              | Manual Trigger               | Check PLL Endpoint                | Polylang Debugging             |
| Check PLL Endpoint      | httpRequest (WordPress API)       | Check Polylang REST API endpoint                   | Get Posts Schema             | None                            | Polylang Debugging             |
| Get API Routes          | httpRequest (WordPress API)       | Get WP REST API root and routes                    | Manual Trigger               | Analyze API Routes                | Polylang Debugging             |
| Analyze API Routes      | code                             | Analyze API namespaces and routes for Polylang    | Get API Routes               | None                            | Polylang Debugging             |
| Analyze Post Structure  | code                             | Analyze a post’s JSON for language metadata        | Get Single Post Details      | None                            | Polylang Debugging             |
| Get Terms               | httpRequest (WordPress API)       | Fetch taxonomy terms                               | Manual Trigger               | None                            | Polylang Debugging             |
| Sticky Note             | stickyNote                       | Annotation for Polylang Debugging block            | None                         | None                            | Polylang Debugging             |
| Weekly Schedule         | scheduleTrigger                  | Scheduled trigger for article generation           | None                         | Generate Topic & Metadata         | Article Generator             |
| Generate Topic & Metadata | chainLlm (LangChain)            | Generate topic and metadata JSON                    | Weekly Schedule, OpenAI Model - Topic1, Parse Topic JSON1 | Generate Article Content         | Article Generator             |
| OpenAI Model - Topic1   | lmChatOpenAi                    | Language model for topic generation                 | None                         | Generate Topic & Metadata          | Article Generator             |
| Parse Topic JSON1       | outputParserStructured          | Parse JSON output from topic generation             | OpenAI Model - Topic1        | Generate Topic & Metadata          | Article Generator             |
| Generate Article Content | agent (LangChain)               | Generate full blog article HTML content             | Generate Topic & Metadata, OpenAI Model - Article1 | Create WordPress Draft            | Article Generator             |
| OpenAI Model - Article1 | lmChatOpenAi                    | Language model for article content generation       | None                         | Generate Article Content           | Article Generator             |
| Create WordPress Draft  | wordpress                      | Create draft post in WordPress                       | Generate Article Content     | Set Yoast SEO Data                | Article Generator             |
| Set Yoast SEO Data      | httpRequest (WordPress API)       | Update post with Yoast SEO meta fields              | Create WordPress Draft       | Generate Featured Image           | Article Generator             |
| Generate Featured Image | openAi (Image Generation)       | Generate featured image with AI                      | Set Yoast SEO Data           | Upload Image to WordPress         | Article Generator             |
| Upload Image to WordPress | httpRequest (WordPress API)     | Upload image media to WordPress                      | Generate Featured Image      | Attach Featured Image             | Article Generator             |
| Attach Featured Image   | httpRequest (WordPress API)       | Attach uploaded image as featured media to post    | Upload Image to WordPress    | None                            | Article Generator             |
| Sticky Note1            | stickyNote                       | Annotation for Article Generator block               | None                         | None                            | Article Generator             |

---

## 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Type: manualTrigger  
   - Purpose: Start Polylang debugging block manually.  

2. **Create HTTP Request Nodes for Polylang Debugging**  
   - Get Recent Posts: GET `https://<yourweb.site>/wp-json/wp/v2/posts?per_page=5` with WordPress API credentials. Enable full response.  
   - Get Single Post Details: GET `https://<yourweb.site>/wp-json/wp/v2/posts/{{ $json.id }}?_embed` with credentials. Dynamically use post ID from Get Recent Posts.  
   - Get All Taxonomies: GET `https://<yourweb.site>/wp-json/wp/v2/taxonomies` with credentials.  
   - Get Posts Schema: OPTIONS `https://<yourweb.site>/wp-json/wp/v2/posts` with credentials and full response.  
   - Check PLL Endpoint: GET `https://<yourweb.site>/wp-json/pll/v1` with credentials. Set option to never error on response.  
   - Get API Routes: GET `https://<yourweb.site>/wp-json` with credentials.  
   - Get Terms: GET `https://<yourweb.site>/wp-json/wp/v2/terms` with credentials, never error option.  

3. **Create Code Nodes for Analysis**  
   - Analyze Post Structure: Paste provided JavaScript code analyzing language fields in post JSON. Input from Get Single Post Details.  
   - Analyze Taxonomies: Paste JavaScript code analyzing taxonomies for language taxonomies. Input from Get All Taxonomies.  
   - Analyze API Routes: Paste JavaScript code analyzing API namespaces and routes for Polylang endpoints. Input from Get API Routes.  

4. **Link Nodes for Polylang Debugging**  
   - Manual Trigger outputs to Get Recent Posts, Get All Taxonomies, Get Posts Schema, Get API Routes.  
   - Get Recent Posts outputs to Get Single Post Details.  
   - Get Single Post Details outputs to Analyze Post Structure.  
   - Get All Taxonomies outputs to Analyze Taxonomies.  
   - Get API Routes outputs to Analyze API Routes.  
   - Get Posts Schema outputs to Check PLL Endpoint.  

5. **Add Sticky Note** titled "Polylang Debugging" near this block for clarity.  

6. **Create Weekly Schedule Trigger Node**  
   - Configure to trigger every Wednesday at 10:00 AM.  

7. **Create OpenAI Language Model Node for Topic Generation (OpenAI Model - Topic1)**  
   - Model: GPT-4.1-mini  
   - Temperature: 0.7  
   - Use OpenAI API credentials.  

8. **Create Output Parser Node (Parse Topic JSON1)**  
   - Use the example JSON schema to parse topic and metadata JSON output from language model.  

9. **Create Chain LLM Node (Generate Topic & Metadata)**  
   - Use prompt as specified: ask for one topic, category, language (en, tr, fa), and metadata.  
   - Set output parser to Parse Topic JSON1.  
   - Link OpenAI Model - Topic1 as language model for this node.  

10. **Create OpenAI Language Model Node for Article Content (OpenAI Model - Article1)**  
    - Model: GPT-4.1-mini  
    - Max Tokens: 2500  
    - Temperature: 0.8  
    - Use OpenAI API credentials.  

11. **Create Agent Node for Article Generation (Generate Article Content)**  
    - Use prompt to generate 1000–1500 words article in HTML format with given structure and SEO guidelines.  
    - Link OpenAI Model - Article1 as language model.  
    - Input variables from Generate Topic & Metadata node output.  

12. **Create WordPress Node (Create WordPress Draft)**  
    - Action: Create post  
    - Set status to draft  
    - Title: from topic metadata title  
    - Slug: from topic metadata slug  
    - Content: from article content output (HTML)  
    - Categories: from topic metadata categoryId  
    - Author ID: 2 (default)  
    - Use WordPress API credentials.  

13. **Create HTTP Request Node (Set Yoast SEO Data)**  
    - POST to `https://<yourweb.site>/wp-json/wp/v2/posts/{{ post_id }}`  
    - Body JSON: set Yoast meta fields `_yoast_wpseo_focuskw`, `_yoast_wpseo_metadesc`, `_yoast_wpseo_title` with corresponding metadata values.  
    - Use WordPress API credentials.  

14. **Create OpenAI Image Generation Node (Generate Featured Image)**  
    - Prompt: cinematic, realistic photo for the blog post title with specified style requirements.  
    - Size: 1792x1024  
    - Style: natural  
    - Quality: standard  
    - Use OpenAI API credentials.  

15. **Create HTTP Request Node (Upload Image to WordPress)**  
    - POST to `https://<yourweb.site>/wp-json/wp/v2/media`  
    - Send binary data from image generation  
    - Headers: Content-Type: image/png; Content-Disposition: attachment; filename=<slug>.png  
    - Use WordPress API credentials.  

16. **Create HTTP Request Node (Attach Featured Image)**  
    - POST to update the post with `featured_media` set to uploaded image ID.  
    - Use WordPress API credentials.  

17. **Link Article Generation Nodes**  
    - Weekly Schedule → Generate Topic & Metadata  
    - Generate Topic & Metadata → Generate Article Content  
    - Generate Article Content → Create WordPress Draft  
    - Create WordPress Draft → Set Yoast SEO Data  
    - Set Yoast SEO Data → Generate Featured Image  
    - Generate Featured Image → Upload Image to WordPress  
    - Upload Image to WordPress → Attach Featured Image  

18. **Add Sticky Note** titled "Article Generator" near this block for clarity.  

---

## 5. General Notes & Resources

| Note Content                                                                                         | Context or Link                                           |
|----------------------------------------------------------------------------------------------------|-----------------------------------------------------------|
| The workflow assumes WordPress REST API with Polylang plugin installed and accessible via OAuth2 or API key. | Ensure WordPress API credentials are correctly configured. |
| OpenAI API usage requires valid API keys with sufficient quota for GPT-4.1-mini and image generation. | See https://platform.openai.com/docs/api-reference        |
| Yoast SEO fields are set via custom meta keys `_yoast_wpseo_focuskw`, `_yoast_wpseo_metadesc`, etc. | Yoast SEO REST API docs: https://developer.yoast.com      |
| Scheduled node triggers weekly on Wednesday at 10:00 AM to automate content creation workflow.      | Adjust schedule as needed per editorial calendar.          |
| The article content generation avoids legal advice and immigration law interpretation per prompt.   | Critical for compliance and liability considerations.      |
| Polylang debugging nodes help adapt the workflow to various multilingual setups and WordPress versions.| Useful for troubleshooting and custom integration.         |

---

**Disclaimer:** The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and publicly accessible.