Automated Blog Post Generator with GPT-4, DALL-E, and Wikipedia for WordPress

https://n8nworkflows.xyz/workflows/automated-blog-post-generator-with-gpt-4--dall-e--and-wikipedia-for-wordpress-9018


# Automated Blog Post Generator with GPT-4, DALL-E, and Wikipedia for WordPress

### 1. Workflow Overview

This workflow automates the creation of a WordPress blog post starting from user-provided keywords. It leverages AI models (GPT-4 and DALL-E) to generate a structured article including title, subtitle, chapters, introduction, conclusions, and a featured image. Wikipedia is used as an auxiliary knowledge source to enrich content accuracy. The workflow is structured into several logical blocks:

- **1.1 User Input Reception:** Collects keywords, number of chapters, and maximum word count from a web form.
- **1.2 Settings Preparation:** Sets static and dynamic parameters including WordPress URL and input values.
- **1.3 Article Structure Generation:** Uses GPT-4 to create the article outline and metadata.
- **1.4 Data Validation:** Checks that all necessary content components are present.
- **1.5 Chapter Processing:** Splits chapters, generates text for each chapter via GPT-4, and merges them.
- **1.6 Article Assembly:** Combines introduction, chapters, and conclusions into the full article HTML.
- **1.7 WordPress Posting:** Posts the article as a draft on WordPress.
- **1.8 Featured Image Creation:** Generates an image via DALL-E, uploads it to WordPress, and sets it as the post’s featured image.
- **1.9 User Feedback:** Responds to the user with success or error messages.

---

### 2. Block-by-Block Analysis

#### 2.1 User Input Reception

- **Overview:** Captures user inputs to trigger post creation.
- **Nodes Involved:** `Form`
- **Node Details:**
  - **Form**  
    - Type: Form Trigger (Webhook)  
    - Configuration: Exposes a form with three fields:
      - Keywords (comma-separated) [required]
      - Number of chapters (dropdown 1-10) [required]
      - Max words count (number) [required]  
    - Input: HTTP webhook triggered by form submission  
    - Output: JSON with form data  
    - Potential Failures: Missing required fields, malformed input  
    - Notes: User initiates the workflow by submitting this form.

#### 2.2 Settings Preparation

- **Overview:** Maps user form data to workflow variables and sets the WordPress URL.
- **Nodes Involved:** `Settings`
- **Node Details:**
  - **Settings**  
    - Type: Set node  
    - Configuration: Assigns static WordPress URL and extracts form inputs (`keywords`, `chapters`, `words`) into variables for downstream use.  
    - Input: Output from `Form` node  
    - Output: JSON with structured parameters  
    - Edge Cases: Incorrect or missing WordPress URL can cause posting failures.

#### 2.3 Article Structure Generation

- **Overview:** Generates the article’s metadata and structure using GPT-4, enriched with Wikipedia data.
- **Nodes Involved:** `Wikipedia`, `Create post title and structure`
- **Node Details:**
  - **Wikipedia**  
    - Type: Wikipedia AI Tool  
    - Role: Provides factual information for GPT-4 prompt enrichment.  
    - Input: Triggered by `Create post title and structure` node  
    - Output: Augmented knowledge for AI generation  
  - **Create post title and structure**  
    - Type: OpenAI GPT-4 (LangChain node)  
    - Configuration: Uses GPT-4 (model: gpt-4-1106-preview) to generate JSON including title, subtitle, introduction (~60 words), conclusions (~60 words), image prompt, and chapters (titles & prompts).  
    - Key Prompts:  
      - Instructs to write SEO-friendly article structure based on keywords and chapter count.  
      - Requires chapters to follow logical flow without repetition.  
      - Uses HTML limited to bold, italic, and lists for formatting.  
      - Requires Wikipedia verification of content if available.  
    - Input: Variables from `Settings` node, Wikipedia knowledge  
    - Output: JSON with complete article structure  
    - Failure Modes: AI misinterpretation, incomplete data, API timeouts.

#### 2.4 Data Validation

- **Overview:** Ensures all required article components are present to continue.
- **Nodes Involved:** `Check data consistency`
- **Node Details:**
  - **Check data consistency**  
    - Type: If node  
    - Checks presence and non-empty values for:  
      - Title  
      - Subtitle  
      - Chapters (array length ≥ 1)  
      - Introduction  
      - Conclusions  
      - Image prompt  
    - Input: Output from `Create post title and structure`  
    - Output:  
      - True branch: proceeds to chapter processing  
      - False branch: triggers error response to user  
    - Edge Cases: Missing or malformed AI output causes early termination.

#### 2.5 Chapter Processing

- **Overview:** Processes each chapter individually by splitting, generating text, and merging results.
- **Nodes Involved:** `Split out chapters`, `Create chapters text`, `Merge chapters title and text`
- **Node Details:**
  - **Split out chapters**  
    - Type: SplitOut node  
    - Splits the chapters array into individual items for parallel processing.  
    - Input: Valid article structure  
    - Output: Individual chapter objects  
  - **Create chapters text**  
    - Type: OpenAI GPT-4 (LangChain node)  
    - Configuration: Uses GPT-4 (model: gpt-4-0125-preview) to generate chapter text based on chapter prompt and related context.  
    - Key Prompts:  
      - Writes ~ (max words - 120) / chapter_count words per chapter.  
      - Uses HTML formatting (bold, italic, lists).  
      - Ensures coherence with previous and following chapters.  
      - Does not include chapter headings or introduction/conclusion.  
    - Input: Individual chapter data from `Split out chapters`  
    - Output: Chapter text  
    - Edge Cases: API failures, incoherent text, exceeding token limits  
  - **Merge chapters title and text**  
    - Type: Merge node (combine by position)  
    - Combines chapter titles with generated text back into a single array.  
    - Input: Outputs from `Split out chapters` (titles) and `Create chapters text` (texts)  
    - Output: Combined chapters for article assembly

#### 2.6 Article Assembly

- **Overview:** Combines introduction, chapters, and conclusions into the final article HTML.
- **Nodes Involved:** `Final article text`
- **Node Details:**
  - **Final article text**  
    - Type: Code (JavaScript)  
    - Function:  
      - Starts with introduction text  
      - Appends each chapter with its title in bold followed by its text, separated by line breaks  
      - Ends with conclusions in bold followed by conclusion text  
    - Input: Merged chapters and article structure  
    - Output: Single HTML string under key `article`  
    - Edge Cases: Improper HTML formatting or missing sections.

#### 2.7 WordPress Posting

- **Overview:** Posts the assembled article as a draft on WordPress.
- **Nodes Involved:** `Post on Wordpress`
- **Node Details:**
  - **Post on Wordpress**  
    - Type: WordPress node  
    - Configuration:  
      - Title from article structure  
      - Content from `Final article text` node  
      - Status set to `draft`  
    - Credentials: WordPress API credentials with posting permissions  
    - Input: Final assembled article  
    - Output: Post metadata including post ID  
    - Edge Cases: Authentication failure, WordPress REST API errors, invalid URL

#### 2.8 Featured Image Creation

- **Overview:** Generates and attaches a featured image to the post.
- **Nodes Involved:** `Generate featured image`, `Upload media`, `Set image ID for the post`
- **Node Details:**
  - **Generate featured image**  
    - Type: OpenAI DALL-E (LangChain node)  
    - Configuration:  
      - Prompt includes article title and AI-generated image prompt  
      - Image size 1792x1024, natural style, HD quality  
    - Input: Article structure  
    - Output: Binary image data  
    - Edge Cases: API quota limits, generation failures  
  - **Upload media**  
    - Type: HTTP Request  
    - Configuration:  
      - POST to WordPress media API endpoint  
      - Sends image as binary with proper headers and authorization  
    - Credentials: WordPress API credentials  
    - Input: Binary image from `Generate featured image`  
    - Output: Media metadata including media ID  
    - Edge Cases: Upload errors, authentication issues  
  - **Set image ID for the post**  
    - Type: HTTP Request  
    - Configuration:  
      - POST to WordPress post endpoint with `featured_media` parameter set to uploaded media ID  
    - Credentials: WordPress API credentials  
    - Input: Post ID from `Post on Wordpress`, media ID from `Upload media`  
    - Output: Updated post metadata  
    - Edge Cases: Incorrect post or media IDs, API errors

#### 2.9 User Feedback

- **Overview:** Provides success or error response back to the form submitter.
- **Nodes Involved:** `Respond: Success`, `Respond: Error`
- **Node Details:**
  - **Respond: Success**  
    - Type: Respond to Webhook  
    - Configuration: Returns JSON confirming draft creation with post title  
    - Triggered after successful post and image upload  
  - **Respond: Error**  
    - Type: Respond to Webhook  
    - Configuration: Returns JSON with error message asking user to retry  
    - Triggered if data validation fails

---

### 3. Summary Table

| Node Name                   | Node Type                         | Functional Role                      | Input Node(s)                | Output Node(s)               | Sticky Note                                                                                         |
|-----------------------------|----------------------------------|------------------------------------|-----------------------------|-----------------------------|---------------------------------------------------------------------------------------------------|
| Form                        | Form Trigger                     | User form input reception           |                             | Settings                    | ## User form; The user triggers the post creation                                                |
| Settings                    | Set                              | Prepare parameters and WordPress URL| Form                        | Create post title and structure | ## Settings; Set the URL of your WordPress here                                                  |
| Wikipedia                   | LangChain Wikipedia tool         | Enrich AI with Wikipedia data       |                             | Create post title and structure (ai_tool) | Wikipedia is used to write the article                                                           |
| Create post title and structure | LangChain OpenAI GPT-4           | Generate article structure and prompts| Settings, Wikipedia          | Check data consistency      | ## Article structure; Starting from the given keywords, generates the article title, subtitle, chapters, and image prompt |
| Check data consistency      | If                               | Validate AI output completeness     | Create post title and structure | Split out chapters; Respond: Error | ## Data check; Checks that the data returned by OpenAI is correct                                |
| Split out chapters          | SplitOut                         | Split chapters array for processing | Check data consistency      | Merge chapters title and text; Create chapters text | ## Chapters split; Splits out chapter contents from the previous node                            |
| Create chapters text        | LangChain OpenAI GPT-4           | Generate text for each chapter      | Split out chapters          | Merge chapters title and text | ## Chapters text; Writes the text for each chapter                                              |
| Merge chapters title and text| Merge                           | Combine chapter titles and texts    | Split out chapters; Create chapters text | Final article text          | ## Content preparation; Merges the content and prepare it before sending it to WordPress         |
| Final article text          | Code                             | Assemble full article HTML          | Merge chapters title and text | Post on Wordpress           |                                                                                                   |
| Post on Wordpress           | WordPress                        | Publish draft post                  | Final article text          | Generate featured image      | ## Draft on WordPress; The article is posted as a draft on WordPress                             |
| Generate featured image     | LangChain OpenAI DALL-E          | Generate cover image                | Post on Wordpress           | Upload media                | ## Featured image; The image is generated with Dall-E, uploaded to WordPress, and then connected to the post as its featured image |
| Upload media                | HTTP Request                    | Upload image to WordPress media library| Generate featured image    | Set image ID for the post    |                                                                                                   |
| Set image ID for the post   | HTTP Request                    | Assign uploaded image as featured media| Upload media               | Respond: Success            |                                                                                                   |
| Respond: Success            | Respond to Webhook               | Confirm successful article creation | Set image ID for the post   |                             | ## User feedback; Final confirmation to the user                                                |
| Respond: Error              | Respond to Webhook               | Return error message on failure    | Check data consistency (false branch) |                             | User is notified to try again since some data is missing                                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Form Trigger node (named `Form`):**  
   - Path: `create-wordpress-post`  
   - Form Title: "Create a WordPress post with AI"  
   - Form Fields:  
     - Keywords (comma-separated), required text  
     - Number of chapters, dropdown (options 1-10), required  
     - Max words count, number, required  
   - Response Mode: Response node

2. **Add a Set node (`Settings`):**  
   - Assign:  
     - `wordpress_url` = string, your WordPress base URL (e.g., https://your-wordpress-url.com/)  
     - `keywords` = expression: `{{$json["Keywords (comma-separated)"]}}`  
     - `chapters` = number: `{{$json["Number of chapters"]}}`  
     - `words` = number: `{{$json["Max words count"]}}`  
   - Connect from `Form`

3. **Add a Wikipedia AI Tool node (`Wikipedia`):**  
   - No parameters required  
   - This node will be invoked by the next OpenAI node

4. **Add an OpenAI GPT-4 node (`Create post title and structure`):**  
   - Model: `gpt-4-1106-preview`  
   - Max tokens: 2048  
   - Messages: Single system/user message with detailed prompt instructing to create:  
     - title, subtitle, introduction (~60 words), conclusions (~60 words), image prompt  
     - chapters array with `title` and `prompt` for each chapter (count from `chapters`)  
     - Use HTML formatting (bold, italic, lists), no markdown  
     - Use Wikipedia data internally for fact-checking if applicable  
   - Set JSON output enabled  
   - Credentials: OpenAI API credentials  
   - Connect `Settings` node to this node’s main input  
   - Connect `Wikipedia` node via AI Tool input of this node

5. **Add an If node (`Check data consistency`):**  
   - Conditions: Check that the following fields are non-empty/non-null in the output JSON:  
     - `message.content.title` (string not empty)  
     - `message.content.subtitle` (string not empty)  
     - `message.content.chapters` (array length ≥ 1)  
     - `message.content.introduction` (string not empty)  
     - `message.content.conclusions` (string not empty)  
     - `message.content.imagePrompt` (string not empty)  
   - Connect from `Create post title and structure`  
   - True branch connects to chapter processing  
   - False branch connects to error response

6. **Add a SplitOut node (`Split out chapters`):**  
   - Field to split out: `message.content.chapters`  
   - Connect from If node’s true branch

7. **Add an OpenAI GPT-4 node (`Create chapters text`):**  
   - Model: `gpt-4-0125-preview`  
   - Max tokens: 2048  
   - Messages: Prompt to write chapter text based on chapter title and prompt, including context from surrounding chapters, using HTML formatting only (bold, italic, lists), no markdown.  
   - Length target: approximately (`words` -120) / `chapters` words per chapter  
   - Credentials: OpenAI API credentials  
   - Connect from `Split out chapters`

8. **Add a Merge node (`Merge chapters title and text`):**  
   - Mode: Combine  
   - Combination mode: Merge by position  
   - Connect inputs from both `Split out chapters` (titles) and `Create chapters text` (texts)

9. **Add a Code node (`Final article text`):**  
   - JavaScript code to concatenate:  
     - Introduction from `Create post title and structure`  
     - Each chapter’s title (bold) and text separated by `<br><br>`  
     - Conclusions (bold) and conclusions text  
   - Output field: `article` with full HTML string  
   - Connect from `Merge chapters title and text`

10. **Add a WordPress node (`Post on Wordpress`):**  
    - Credentials: WordPress API with posting rights  
    - Title: Expression from `Create post title and structure`: `{{$json.message.content.title}}`  
    - Content: Expression from `Final article text`: `{{$json.article}}`  
    - Status: Draft  
    - Connect from `Final article text`

11. **Add an OpenAI DALL-E node (`Generate featured image`):**  
    - Resource: Image generation  
    - Prompt: Combine article title and image prompt from `Create post title and structure` with photographic style instructions  
    - Image size: 1792x1024  
    - Style: Natural  
    - Quality: HD  
    - Credentials: OpenAI API  
    - Connect from `Post on Wordpress`

12. **Add HTTP Request node (`Upload media`):**  
    - Method: POST  
    - URL: `{wordpress_url}/wp-json/wp/v2/media` (replace with your WordPress URL)  
    - Content type: binary data  
    - Send headers:  
      - `Content-Disposition: attachment; filename="example.jpg"`  
    - Authentication: Use WordPress API credentials  
    - Input data field name: `data` (binary image from previous node)  
    - Connect from `Generate featured image`

13. **Add HTTP Request node (`Set image ID for the post`):**  
    - Method: POST  
    - URL: `{wordpress_url}/wp-json/wp/v2/posts/{post_id}` where `post_id` is from `Post on Wordpress` output JSON  
    - Query parameter: `featured_media` = ID from `Upload media` response JSON  
    - Authentication: WordPress API credentials  
    - Connect from `Upload media`

14. **Add Respond to Webhook node (`Respond: Success`):**  
    - Response type: JSON  
    - Body: `{"formSubmittedText":"The article {{$json.title.rendered}} was correctly created as a draft on WordPress!"}`  
    - Connect from `Set image ID for the post`

15. **Add Respond to Webhook node (`Respond: Error`):**  
    - Response type: JSON  
    - Body: `{"formSubmittedText":"There was a problem creating the article, please refresh the form and try again!"}`  
    - Connect from `Check data consistency` false branch

---

### 5. General Notes & Resources

| Note Content                                                                                                  | Context or Link                                                                                          |
|---------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| The article is posted as a draft on WordPress and not published directly to allow manual review.              | WordPress Posting Block                                                                                   |
| Image generation uses DALL-E via OpenAI with specific photographic style instructions for realism and quality. | Featured Image Block                                                                                       |
| Wikipedia data is integrated as a knowledge tool to improve factual accuracy of generated content.            | AI Content Generation Block                                                                               |
| The workflow is designed to maintain coherent, fluent article flow avoiding repetition between chapters.      | Chapter Generation and Processing                                                                         |
| Ensure WordPress API credentials have required scopes for posting and media upload operations.                | WordPress Credential Setup                                                                                 |
| OpenAI API credentials must have access to GPT-4 and DALL-E models and sufficient quota for prompt processing. | OpenAI Credential Setup                                                                                     |
| The form trigger node exposes a public webhook; secure this endpoint as needed to prevent unauthorized use.   | User Input Reception                                                                                       |
| Recommended max tokens are set conservatively to avoid truncation in GPT-4 calls; adjust based on need.        | AI Nodes Configuration                                                                                     |
| HTML formatting in generated content is limited to bold, italic, and lists to maintain WordPress compatibility.| Formatting Guidelines                                                                                       |
| API error handling is basic; consider adding retries or error logging for production environments.             | Workflow Robustness                                                                                         |