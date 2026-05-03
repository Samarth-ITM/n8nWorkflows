Multilingual RSS to WordPress Publishing with OpenAI, ACF, and AI-Generated Images

https://n8nworkflows.xyz/workflows/multilingual-rss-to-wordpress-publishing-with-openai--acf--and-ai-generated-images-11262


# Multilingual RSS to WordPress Publishing with OpenAI, ACF, and AI-Generated Images

### 1. Workflow Overview

This workflow automates the process of ingesting news articles from an RSS feed, extracting and rewriting their content, translating them into multiple languages, generating AI-based featured images, and publishing the multilingual articles on a WordPress website with Advanced Custom Fields (ACF) integration.

The workflow is structured into the following logical blocks:

- **1.1 Input Reception and Scraping:** Fetches new RSS feed items hourly, assigns URLs, and scrapes the full HTML content of each article.
- **1.2 Article Extraction and Rewriting:** Uses AI (OpenAI) to extract the main article title and body from raw HTML, then rewrites the article in a predefined main language adapting style and formatting.
- **1.3 Multilingual Translation:** Translates the rewritten article into additional target languages, preserving HTML formatting and factual accuracy.
- **1.4 Data Aggregation and Structuring:** Aggregates translated articles and restructures data for WordPress ACF fields, preparing multilingual content.
- **1.5 AI-Based Featured Image Generation:** Generates a unique image prompt, requests an AI image generation API, downloads the image, and uploads it to WordPress, capturing the media ID.
- **1.6 WordPress Publishing:** Combines article data and image ID, then creates a multilingual WordPress post with the featured image and ACF custom fields.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and Scraping

- **Overview:**  
  This block triggers the workflow every hour based on the RSS feed, assigns the article URL to a field, and scrapes the full HTML content of the linked article using an external scraping API.

- **Nodes Involved:**  
  - RSS Feed Trigger  
  - Assign URL  
  - Scrape Page  
  - Detect and Extract Article from Page HTML  
  - Assign and Multilingual Prompt  
  - Sticky Note1

- **Node Details:**

  - **RSS Feed Trigger**  
    - *Type:* RSS Feed Read Trigger  
    - *Role:* Periodically polls the specified RSS feed (Al Jazeera) every hour to detect new articles.  
    - *Configuration:* Feed URL set to "https://www.aljazeera.com/xml/rss/all.xml"; polling every hour.  
    - *Inputs:* None (trigger)  
    - *Outputs:* Emits new RSS items  
    - *Failures:* Network errors, feed format errors, rate limits.

  - **Assign URL**  
    - *Type:* Set Node  
    - *Role:* Extracts and assigns the article URL from the RSS item for downstream use.  
    - *Configuration:* Sets a new field `link` equal to the RSS item's `link`.  
    - *Inputs:* RSS Feed Trigger output  
    - *Outputs:* JSON enriched with `link` field.

  - **Scrape Page**  
    - *Type:* HTTP Request  
    - *Role:* Posts the article URL to Firecrawl API to scrape the full HTML content.  
    - *Configuration:* POST to "https://api.firecrawl.dev/v1/scrape" with JSON body containing the URL, request for HTML format, max age 1 hour, no main content filtering, and waiting 3 seconds for dynamic content. Auth: Generic HTTP Bearer (credential must be set).  
    - *Inputs:* JSON with `link` field  
    - *Outputs:* Scraped HTML data  
    - *Failures:* Auth errors, API downtime, invalid URL, timeouts.

  - **Detect and Extract Article from Page HTML**  
    - *Type:* OpenAI (Langchain)  
    - *Role:* Uses AI to parse scraped HTML and extract the main article title and clean body text.  
    - *Configuration:* Uses gpt-4o-mini model with JSON schema output for title and message. System prompt specifies detailed extraction rules for title and body, cleaning HTML tags, removing ads, etc.  
    - *Inputs:* Scrape Page output JSON with field `data.html`  
    - *Outputs:* JSON with `"title"` and `"content"` fields extracted from raw HTML.  
    - *Failures:* AI API errors, incomplete extraction if HTML is malformed.

  - **Assign and Multilingual Prompt**  
    - *Type:* Set Node  
    - *Role:* Assigns extracted article fields to variables and sets default language and country.  
    - *Configuration:* Sets `original_title` and `original_content` from extraction output; sets `default_language` (hardcoded "german") and `country` ("Germany").  
    - *Inputs:* Extracted article output  
    - *Outputs:* JSON with assigned fields for downstream rewriting and translation.

  - **Sticky Note1**  
    - *Role:* Documentation node explaining this block’s purpose: scraping and extracting article text and title.

---

#### 1.2 Article Extraction and Rewriting

- **Overview:**  
  This block rewrites the extracted article in the main language, adapting style and formatting using OpenAI. It prepares the content for translation.

- **Nodes Involved:**  
  - Rewrite article in default language  
  - Get rewrited article  
  - Sticky Note7

- **Node Details:**

  - **Rewrite article in default language**  
    - *Type:* OpenAI (Langchain)  
    - *Role:* Rewrites the original article content into the main language (German), applying journalistic style, adding context, and formatting with HTML tags.  
    - *Configuration:* Uses gpt-4o-mini with JSON schema output of title and message. System prompt instructs detailed rewriting with HTML, no newline characters, SEO considerations.  
    - *Inputs:* `original_content`, `default_language`, `country` from Assign and Multilingual Prompt  
    - *Outputs:* JSON rewritten article (`title` and `content`) in main language.  
    - *Failures:* AI API errors, prompt misconfiguration.

  - **Get rewrited article**  
    - *Type:* Set Node  
    - *Role:* Extracts rewritten article JSON, sets `news` object and specifies `default_language` and array of `other_languages` to translate into (`["english","italian"]` hardcoded).  
    - *Configuration:* Assigns fields based on previous node output.  
    - *Inputs:* Rewrite article output  
    - *Outputs:* Structured JSON for translation input.

  - **Sticky Note7**  
    - *Role:* Describes this block's purpose: rewriting articles in the main language adapting style and adding HTML formatting.

---

#### 1.3 Multilingual Translation

- **Overview:**  
  Translates the rewritten main language article into additional target languages, preserving HTML structure and factual accuracy using OpenAI.

- **Nodes Involved:**  
  - Split Out Translate Languages  
  - Translate news  
  - Clean Data  
  - Aggregate  
  - Sticky Note2

- **Node Details:**

  - **Split Out Translate Languages**  
    - *Type:* SplitOut  
    - *Role:* Splits the array of target languages into separate workflow items for parallel translation processing.  
    - *Configuration:* Field to split: `other_languages` (e.g., "english", "italian").  
    - *Inputs:* Get rewrited article output  
    - *Outputs:* One item per language.

  - **Translate news**  
    - *Type:* OpenAI (Langchain)  
    - *Role:* Translates the article title and content into one target language preserving HTML tags, formatting, and factual accuracy.  
    - *Configuration:* Uses gpt-4o-mini model; system prompt specifies translation rules for technology news, preserving formatting and no modification of HTML tags.  
    - *Inputs:* JSON with `news` (title, message) and `other_languages` (current target language)  
    - *Outputs:* Translated article JSON with `title` and `content`.  
    - *Failures:* API errors, complexity in maintaining HTML structure.

  - **Clean Data**  
    - *Type:* Set Node  
    - *Role:* Extracts translated news content and language code from translation output for aggregation.  
    - *Configuration:* Sets `translated_news` from translation output, `translated_language` from Split Out node.  
    - *Inputs:* Translate news output  
    - *Outputs:* Cleaned JSON for aggregation.

  - **Aggregate**  
    - *Type:* Aggregate  
    - *Role:* Aggregates all translated article items back into a single item collection for further processing.  
    - *Configuration:* Aggregate all item data into one array.  
    - *Inputs:* Clean Data outputs for each language  
    - *Outputs:* Single item containing an array of translations.

  - **Sticky Note2**  
    - *Role:* Explains that this block handles accurate translations preserving content and formatting for multiple languages.

---

#### 1.4 Data Aggregation and Structuring

- **Overview:**  
  Combines the original rewritten article with translations, restructures data into Advanced Custom Fields (ACF) format for WordPress, preparing for publishing.

- **Nodes Involved:**  
  - Code in JavaScript  
  - Merge Article with Translations  
  - Sticky Note9

- **Node Details:**

  - **Merge Article with Translations**  
    - *Type:* Merge  
    - *Role:* Combines rewritten main language article with translated articles by position into one item for unified processing.  
    - *Configuration:* Combine mode, including unpaired items.  
    - *Inputs:* Get rewrited article and Aggregate outputs  
    - *Outputs:* Combined article and translations.

  - **Code in JavaScript**  
    - *Type:* Code  
    - *Role:* Transforms aggregated translation array into an `acf` object with keys like `title_language` and `content_language` for each language, matching WordPress ACF field naming.  
    - *Code Logic:* Iterates over each translation, assigns content and title to `acf` object properties keyed by language (e.g. `title_english`).  
    - *Inputs:* Aggregate node output  
    - *Outputs:* JSON containing `acf` object with multilingual fields.

  - **Sticky Note9**  
    - *Role:* Indicates data transformation for WordPress ACF fields and merging multilingual content.

---

#### 1.5 AI-Based Featured Image Generation

- **Overview:**  
  Generates a unique featured image prompt based on the article, sends it to an AI image generation service, downloads the resulting image, and uploads it to WordPress media library.

- **Nodes Involved:**  
  - Generate Image Prompt  
  - Generate Featured Image  
  - Download Image  
  - Upload image to wordpress  
  - Get Image ID  
  - Merge image and articles variables  
  - Sticky Note3

- **Node Details:**

  - **Generate Image Prompt**  
    - *Type:* Code  
    - *Role:* Constructs a prompt JSON instructing the image generation model to create a clean, minimalist, professional featured image relevant to the article title and main language, explicitly excluding text or words in the image.  
    - *Inputs:* Article title and default language  
    - *Outputs:* JSON payload for image generation API.

  - **Generate Featured Image**  
    - *Type:* HTTP Request  
    - *Role:* Sends the prompt to Replicate.com API for image generation, waits for synchronous completion (`Prefer: wait` header).  
    - *Configuration:* POST to Replicate API endpoint with authentication header; content type JSON.  
    - *Inputs:* JSON prompt from previous node  
    - *Outputs:* Response containing image URL and ID.  
    - *Failures:* API key/auth errors, rate limits, generation failures.

  - **Download Image**  
    - *Type:* HTTP Request  
    - *Role:* Downloads the generated image file from the URL returned by the image generation API.  
    - *Configuration:* Configured to receive binary file response.  
    - *Inputs:* Image URL from Generate Featured Image  
    - *Outputs:* Binary image data.

  - **Upload image to wordpress**  
    - *Type:* HTTP Request  
    - *Role:* Uploads the downloaded image binary to WordPress media library via REST API.  
    - *Configuration:* POST to `/wp-json/wp/v2/media` with binary data, content-disposition and content-type headers for `image.webp`. Auth via predefined WordPress API credentials.  
    - *Inputs:* Binary image data  
    - *Outputs:* JSON response with media ID.

  - **Get Image ID**  
    - *Type:* Set Node  
    - *Role:* Extracts the WordPress media ID from upload response to associate as featured image.  
    - *Inputs:* Upload image to wordpress output  
    - *Outputs:* JSON with `image_id` field.

  - **Merge image and articles variables**  
    - *Type:* Merge  
    - *Role:* Combines article content and featured image ID data for final publishing.  
    - *Inputs:* Get Image ID and Merge Article with Translations outputs  
    - *Outputs:* JSON containing article content, translations, ACF fields, and image ID.

  - **Sticky Note3**  
    - *Role:* Describes this block’s function of generating and uploading featured images, customizable via prompt.

---

#### 1.6 WordPress Publishing

- **Overview:**  
  Publishes the final multilingual article on WordPress with the featured image and ACF multilingual fields set.

- **Nodes Involved:**  
  - Publish article and translations  
  - Sticky Note4

- **Node Details:**

  - **Publish article and translations**  
    - *Type:* HTTP Request  
    - *Role:* Sends a POST request to the WordPress REST API to create a new post with title, content, featured image, categories, tags, post format, and ACF fields containing multilingual content.  
    - *Configuration:* URL points to WordPress `/wp-json/wp/v2/posts`. Auth uses predefined WordPress OAuth2 or API credentials. Post status is `publish` (can be changed to `draft` for testing).  
    - *Inputs:* Combined JSON from Merge image and articles variables  
    - *Outputs:* WordPress API response with new post details.  
    - *Failures:* Auth errors, validation errors, connectivity issues.

  - **Sticky Note4**  
    - *Role:* Explains this block’s role in publishing multilingual posts with featured images on WordPress.

---

### 3. Summary Table

| Node Name                      | Node Type                           | Functional Role                             | Input Node(s)                              | Output Node(s)                      | Sticky Note                                                                                                                             |
|--------------------------------|-----------------------------------|---------------------------------------------|--------------------------------------------|-----------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------|
| RSS Feed Trigger               | RSS Feed Read Trigger              | Triggers workflow hourly on new feed items | None                                       | Assign URL                        | ## How it works The workflow checks an RSS feed every hour, scrapes each article, and uses OpenAI to extract the main body content...   |
| Assign URL                    | Set                               | Assigns article URL from RSS item           | RSS Feed Trigger                           | Scrape Page                      |                                                                                                                                         |
| Scrape Page                   | HTTP Request                      | Scrapes full HTML of article page           | Assign URL                                | Detect and Extract Article from Page HTML | ## Scrape, extract article and assign languages This section scrapes the page content and uses OpenAI to extract the article text...    |
| Detect and Extract Article from Page HTML | OpenAI (Langchain)               | Extracts article title and body from HTML   | Scrape Page                               | Assign and Multilingual Prompt    |                                                                                                                                         |
| Assign and Multilingual Prompt | Set                               | Assigns extracted content and language info | Detect and Extract Article from Page HTML | Rewrite article in default language |                                                                                                                                         |
| Rewrite article in default language | OpenAI (Langchain)               | Rewrites article in main language with style | Assign and Multilingual Prompt             | Get rewrited article             | ## Rewriting in your main language This section rewrites the articles in the main language you specified earlier...                     |
| Get rewrited article          | Set                               | Sets news object and target languages       | Rewrite article in default language       | Split Out Translate Languages, Merge Article with Translations |                                                                                                                                         |
| Split Out Translate Languages | SplitOut                         | Splits other_languages array for translation | Get rewrited article                      | Translate news                   |                                                                                                                                         |
| Translate news                | OpenAI (Langchain)               | Translates article to each target language  | Split Out Translate Languages              | Clean Data                      | ## Multilingual Processing This section translates the article body into the languages you specified...                                  |
| Clean Data                   | Set                               | Extracts translation results and language   | Translate news                           | Aggregate                       |                                                                                                                                         |
| Aggregate                   | Aggregate                         | Aggregates all translations into array      | Clean Data                              | Code in JavaScript               |                                                                                                                                         |
| Code in JavaScript           | Code                              | Converts translations to ACF field format   | Aggregate                               | Merge Article with Translations  |                                                                                                                                         |
| Merge Article with Translations | Merge                            | Combines rewritten article with translations | Get rewrited article, Code in JavaScript  | Generate Image Prompt, Merge image and articles variables |                                                                                                                                         |
| Generate Image Prompt        | Code                              | Creates prompt for AI image generation      | Merge Article with Translations           | Generate Featured Image          | ## Featured Image Generation Creates a unique featured image for each article in your preferred style...                                 |
| Generate Featured Image      | HTTP Request                      | Sends prompt to image generation API        | Generate Image Prompt                      | Download Image                  |                                                                                                                                         |
| Download Image              | HTTP Request                      | Downloads generated image file               | Generate Featured Image                    | Upload image to wordpress       |                                                                                                                                         |
| Upload image to wordpress    | HTTP Request                      | Uploads image to WordPress media library     | Download Image                           | Get Image ID                   |                                                                                                                                         |
| Get Image ID                | Set                               | Extracts media ID from upload response       | Upload image to wordpress                 | Merge image and articles variables |                                                                                                                                         |
| Merge image and articles variables | Merge                            | Combines article data and image ID           | Get Image ID, Merge Article with Translations | Publish article and translations |                                                                                                                                         |
| Publish article and translations | HTTP Request                      | Creates WordPress post with featured image and multilingual content | Merge image and articles variables         | None                          | ## WordPress Publishing Creates the final post with a featured image and multilingual fields...                                          |
| Sticky Note1                | Sticky Note                      | Documentation for scraping and extraction   | None                                     | None                           | ## Scrape, extract article and assign languages This section scrapes the page content and uses OpenAI to extract the article text...    |
| Sticky Note2                | Sticky Note                      | Documentation for translation block          | None                                     | None                           | ## Multilingual Processing This section translates the article body into the languages you specified...                                  |
| Sticky Note3                | Sticky Note                      | Documentation for image generation            | None                                     | None                           | ## Featured Image Generation Creates a unique featured image for each article in your preferred style...                                 |
| Sticky Note4                | Sticky Note                      | Documentation for publishing block            | None                                     | None                           | ## WordPress Publishing Creates the final post with a featured image and multilingual fields...                                          |
| Sticky Note7                | Sticky Note                      | Documentation for rewriting block             | None                                     | None                           | ## Rewriting in your main language This section rewrites the articles in the main language you specified earlier...                     |
| Sticky Note6                | Sticky Note                      | Overall workflow explanation and setup steps | None                                     | None                           | ## How it works The workflow checks an RSS feed every hour, scrapes each article, and uses OpenAI to identify the main body content...   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create RSS Feed Trigger node:**  
   - Type: RSS Feed Read Trigger  
   - Parameters: Set feed URL to the RSS source (e.g., `https://www.aljazeera.com/xml/rss/all.xml`)  
   - Poll times: Every hour  
   - No credentials needed.

2. **Add Set node "Assign URL":**  
   - Add field `link` with value `{{$json.link}}` from RSS item.  
   - Connect RSS Trigger → Assign URL.

3. **Add HTTP Request node "Scrape Page":**  
   - Method: POST  
   - URL: `https://api.firecrawl.dev/v1/scrape`  
   - Body (JSON):  
     ```json
     {
       "url": "{{$json.link}}",
       "formats": ["html"],
       "maxAge": 3600000,
       "onlyMainContent": false,
       "waitFor": 3000
     }
     ```  
   - Authentication: Generic HTTP Bearer (configure Firecrawl API token)  
   - Connect Assign URL → Scrape Page.

4. **Add OpenAI node "Detect and Extract Article from Page HTML":**  
   - Model: GPT-4o-mini  
   - Response format: JSON schema with `title` and `message` fields.  
   - System prompt: Use extraction instructions to parse HTML and return clean article title & body.  
   - Input: `{{$json.data.html}}` from Scrape Page  
   - Connect Scrape Page → Detect and Extract Article.

5. **Add Set node "Assign and Multilingual Prompt":**  
   - Assign fields:  
     - `original_title` = `{{$json.output[0].content[0].text.title}}`  
     - `original_content` = `{{$json.output[0].content[0].text.message}}`  
     - `default_language` = `"german"` (or your main language)  
     - `country` = `"Germany"` (or your country)  
   - Connect Detect and Extract Article → Assign and Multilingual Prompt.

6. **Add OpenAI node "Rewrite article in default language":**  
   - Model: GPT-4o-mini  
   - Response JSON schema: `title` and `message`  
   - System prompt: Rewrites article in main language with journalistic style, HTML formatting, no newlines, SEO optimized.  
   - Input: `original_content`, `default_language`, `country` from previous node  
   - Connect Assign and Multilingual Prompt → Rewrite article.

7. **Add Set node "Get rewrited article":**  
   - Assign:  
     - `news` = `{{$json.output[0].content[0].text}}` (rewritten article)  
     - `default_language` = from Assign and Multilingual Prompt  
     - `other_languages` = `["english", "italian"]` (adjust as needed)  
   - Connect Rewrite article → Get rewrited article.

8. **Add SplitOut node "Split Out Translate Languages":**  
   - Split field: `other_languages`  
   - Connect Get rewrited article → Split Out Translate Languages.

9. **Add OpenAI node "Translate news":**  
   - Model: GPT-4o-mini  
   - System prompt: Translate article into target language preserving HTML and factual content for tech news.  
   - Input: `news` and current language from Split Out  
   - Connect Split Out Translate Languages → Translate news.

10. **Add Set node "Clean Data":**  
    - Assign:  
      - `translated_news` = `{{$json.output[0].content[0].text}}`  
      - `translated_language` = `{{$json.other_languages}}` (current language)  
    - Connect Translate news → Clean Data.

11. **Add Aggregate node "Aggregate":**  
    - Aggregate all item data into one array to combine translations.  
    - Connect Clean Data → Aggregate.

12. **Add Code node "Code in JavaScript":**  
    - Script: Iterate over aggregated translations and build an `acf` object with keys `title_language` and `content_language`.  
    - Connect Aggregate → Code in JavaScript.

13. **Add Merge node "Merge Article with Translations":**  
    - Combine mode by position  
    - Inputs:  
      - From Get rewrited article (rewritten main article)  
      - From Code in JavaScript (translations formatted)  
    - Connect Get rewrited article and Code in JavaScript → Merge Article with Translations.

14. **Add Code node "Generate Image Prompt":**  
    - Script: Construct JSON prompt for AI image generation with article title and default language, exclude text in image, specify style, resolution, aspect ratio, and seed.  
    - Connect Merge Article with Translations → Generate Image Prompt.

15. **Add HTTP Request node "Generate Featured Image":**  
    - POST to Replicate.com AI image generation endpoint  
    - Body: JSON prompt from previous node  
    - Headers: Content-Type: application/json, Prefer: wait  
    - Auth: HTTP Header Auth with Replicate API key  
    - Connect Generate Image Prompt → Generate Featured Image.

16. **Add HTTP Request node "Download Image":**  
    - GET or POST to URL returned by image generation API  
    - Response format: File (binary)  
    - Connect Generate Featured Image → Download Image.

17. **Add HTTP Request node "Upload image to wordpress":**  
    - POST to WordPress `/wp-json/wp/v2/media` endpoint  
    - Body: binary image data from Download Image  
    - Headers: Content-Disposition (attachment; filename=image.webp), Content-Type (image/webp)  
    - Auth: WordPress API credentials (OAuth2 or Application Password)  
    - Connect Download Image → Upload image to wordpress.

18. **Add Set node "Get Image ID":**  
    - Extract WordPress media ID from upload response JSON  
    - Connect Upload image to wordpress → Get Image ID.

19. **Add Merge node "Merge image and articles variables":**  
    - Combine mode including unpaired items by position  
    - Inputs:  
      - Get Image ID (image info)  
      - Merge Article with Translations (article content and translations)  
    - Connect Get Image ID and Merge Article with Translations → Merge image and articles variables.

20. **Add HTTP Request node "Publish article and translations":**  
    - POST to WordPress `/wp-json/wp/v2/posts`  
    - Body (JSON):  
      ```json
      {
        "title": {{ JSON.stringify($json.news.title) }},
        "content": {{ JSON.stringify($json.news.message) }},
        "status": "publish",
        "categories": 3,
        "featured_media": {{ $json.image_id }},
        "tags": [],
        "format": "standard",
        "acf": {{ JSON.stringify($json.acf, null, 2) }}
      }
      ```  
    - Auth: WordPress API credentials  
    - Connect Merge image and articles variables → Publish article and translations.

21. **Add Sticky Notes** at appropriate points to document blocks as seen in the original workflow.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                  | Context or Link                                                                                          |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| The workflow assumes a WordPress website with Advanced Custom Fields (ACF) plugin installed and custom fields created for each language, e.g., `title_english`, `content_english`, `title_italian`, `content_italian`.                                                                                                                                        | https://www.advancedcustomfields.com/                                                                   |
| Setup requires credentials for Firecrawl (scraping), OpenAI API, WordPress REST API (OAuth2 or Application Password), and Replicate.com (AI image generation).                                                                                                                                                                                              | Credentials setup in n8n credentials manager                                                            |
| RSS feed URL can be customized to any valid XML feed; example used is Al Jazeera’s global news feed.                                                                                                                                                                                                                                                        | Example: https://www.aljazeera.com/xml/rss/all.xml                                                      |
| The OpenAI prompts are optimized for technology news portals; customize prompts to suit your domain or style preferences.                                                                                                                                                                                                                                   | Prompts embedded inside OpenAI nodes                                                                    |
| Image generation prompt explicitly avoids text in images to maintain visual cleanness and professional style; adjust prompt text in "Generate Image Prompt" node to customize style.                                                                                                                                                                        | Replicate.com model used: black-forest-labs/flux-krea-dev                                              |
| The final posts are published with status `publish`; to test workflow, change `"status"` to `"draft"` in the Publish node.                                                                                                                                                                                                                                   | WordPress post status field                                                                              |

---

*Disclaimer:* The text provided is exclusively from an automated workflow created using n8n, an integration and automation tool. This processing strictly adheres to existing content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.