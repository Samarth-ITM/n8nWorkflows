Generate & Publish SEO Blog Posts to Blogger using OpenRouter AI & Mediastack News

https://n8nworkflows.xyz/workflows/generate---publish-seo-blog-posts-to-blogger-using-openrouter-ai---mediastack-news-6015


# Generate & Publish SEO Blog Posts to Blogger using OpenRouter AI & Mediastack News

### 1. Workflow Overview

This n8n workflow automates the generation and publication of SEO-optimized blog posts on Blogger by leveraging real-time technology news and AI content generation. It is designed for content creators, digital marketers, and SEO specialists who want to streamline blog post creation using current news data and advanced AI writing capabilities.

The workflow is logically divided into the following blocks:

- **1.1 Schedule Trigger & News Retrieval:** Periodically triggers the workflow and fetches the latest technology news via the Mediastack News API.
- **1.2 Content Processing & Title Generation:** Uses AI to generate a concise blog post title, slug, and meta description from the fetched news.
- **1.3 Image Generation & Notification:** Retrieves relevant images from Pexels API based on the news title and sends a Telegram notification with image previews.
- **1.4 SEO Blog Post Writing:** Employs a LangChain AI Agent with OpenRouter to compose a 1,000-word SEO-optimized blog post using the generated metadata and images.
- **1.5 Content Cleanup & Publishing:** Cleans up the AI-generated HTML content, publishes the blog post to Blogger via Google API, and sends a success notification via Telegram.

---

### 2. Block-by-Block Analysis

#### 2.1 Schedule Trigger & News Retrieval

- **Overview:** This block initiates the workflow at scheduled intervals and fetches the latest technology news from Mediastack.
- **Nodes Involved:** `Schedule Trigger`, `Mediastack News`
- **Node Details:**

  - **Schedule Trigger**
    - Type: Schedule Trigger
    - Role: Starts workflow execution every minute.
    - Configuration: Interval set to every 1 minute.
    - Inputs: None
    - Outputs: Mediastack News
    - Potential Failures: Scheduler misconfiguration or system downtime.

  - **Mediastack News**
    - Type: HTTP Request
    - Role: Fetches the latest technology news (1 article, English) from Mediastack API.
    - Configuration: GET request to `https://api.mediastack.com/v1/news?categories=technology&limit=1&languages=en`
    - Authentication: HTTP Query Authentication using API key credential.
    - Inputs: Schedule Trigger
    - Outputs: Genarate image
    - Potential Failures: API key invalid, rate limiting, network issues.

---

#### 2.2 Content Processing & Title Generation

- **Overview:** Extracts the news title and uses an AI agent to generate a slug, blog post title, and meta description optimized for recruitment or HR-related SEO.
- **Nodes Involved:** `AI Agent`, `Parsing`
- **Node Details:**

  - **AI Agent**
    - Type: LangChain AI Agent
    - Role: Creates slug, title, and meta description JSON from the news title.
    - Configuration: Prompt includes detailed guidelines for slug, title, and meta description formatting, focusing on recruitment/HR keywords.
    - Inputs: Output from `Send a text message1` (image notification)
    - Outputs: Parsing
    - Potential Failures: AI service timeout, prompt interpretation errors.

  - **Parsing**
    - Type: Code (JavaScript)
    - Role: Parses AI Agent JSON string output into a JSON object for downstream use.
    - Configuration: Cleans markdown JSON code blocks and parses JSON.
    - Inputs: AI Agent
    - Outputs: Copywriter AI Agent
    - Potential Failures: JSON parse errors if AI output is malformed.

---

#### 2.3 Image Generation & Notification

- **Overview:** Searches for relevant images on Pexels using the news title and notifies via Telegram with image URLs.
- **Nodes Involved:** `Genarate image`, `Send a text message1`
- **Node Details:**

  - **Genarate image**
    - Type: HTTP Request
    - Role: Searches Pexels API for 2 images based on the news title.
    - Configuration: GET request to `https://api.pexels.com/v1/search` with query parameter set to news title and `per_page=2`.
    - Authentication: API Key via HTTP Header Authentication.
    - Inputs: Mediastack News
    - Outputs: Send a text message1
    - Potential Failures: API key invalid, quota exceeded, network issues.

  - **Send a text message1**
    - Type: Telegram Node
    - Role: Sends a Telegram message displaying URLs of the generated images.
    - Configuration: Uses Telegram API credentials, sends message with image URLs in chat identified by `$vars.telegramchatid`.
    - Inputs: Genarate image
    - Outputs: AI Agent
    - Potential Failures: Telegram API downtime, invalid chat ID or token.

---

#### 2.4 SEO Blog Post Writing

- **Overview:** Uses an advanced AI agent (LangChain with OpenRouter Chat Model) to generate a comprehensive, SEO-optimized blog post using structured instructions and metadata.
- **Nodes Involved:** `Copywriter AI Agent`, `OpenRouter Chat Model2`, `Cleanup HTML `
- **Node Details:**

  - **Copywriter AI Agent**
    - Type: LangChain AI Agent
    - Role: Generates a 1000-word SEO blog post HTML content with detailed structure including keywords, headings, images, FAQs, etc.
    - Configuration: Complex prompt referencing parsed title and meta keywords; instructs use of HTML links and semantic SEO strategies.
    - Inputs: Parsing
    - Outputs: Cleanup HTML
    - AI Model: Uses OpenRouter Chat Model2 as language model.
    - Potential Failures: AI service rate limits, incomplete content generation, prompt errors.

  - **Cleanup HTML**
    - Type: Set Node
    - Role: Cleans up AI-generated content by removing markdown code block markers from the HTML output.
    - Configuration: Replaces all occurrences of "```html" and "```" in the content string.
    - Inputs: Copywriter AI Agent
    - Outputs: HTTP Request (Publish)
    - Potential Failures: String replacement errors, empty content.

---

#### 2.5 Content Publishing & Final Notification

- **Overview:** Publishes the cleaned SEO blog post to Blogger via Google API and sends a Telegram confirmation message.
- **Nodes Involved:** `HTTP Request`, `Send a text message`
- **Node Details:**

  - **HTTP Request (Publish)**
    - Type: HTTP Request
    - Role: Posts the blog content to Blogger using Blogger API.
    - Configuration: POST request to `https://www.googleapis.com/blogger/v3/blogs/$vars.bloggerid/posts` with JSON body containing blog ID, title, and cleaned content.
    - Authentication: OAuth2 using Google API credentials.
    - Inputs: Cleanup HTML
    - Outputs: Send a text message
    - Potential Failures: OAuth token expiry, API quota exceeded, malformed content errors.

  - **Send a text message**
    - Type: Telegram Node
    - Role: Sends a Telegram message confirming successful blog post creation with timestamp and post title.
    - Configuration: Uses Telegram API credentials, sends message to chat identified by `$vars.telegramchatid`.
    - Inputs: HTTP Request
    - Outputs: None
    - Potential Failures: Telegram API failures, invalid chat ID.

---

### 3. Summary Table

| Node Name            | Node Type                         | Functional Role                               | Input Node(s)           | Output Node(s)          | Sticky Note                                        |
|----------------------|----------------------------------|----------------------------------------------|-------------------------|-------------------------|---------------------------------------------------|
| Schedule Trigger      | Schedule Trigger                 | Initiates workflow every minute               |                         | Mediastack News         |                                                   |
| Mediastack News       | HTTP Request                    | Fetches latest technology news                 | Schedule Trigger         | Genarate image          |                                                   |
| Genarate image        | HTTP Request                    | Retrieves 2 images from Pexels API             | Mediastack News          | Send a text message1    |                                                   |
| Send a text message1  | Telegram                        | Sends Telegram message with image URLs        | Genarate image           | AI Agent                |                                                   |
| AI Agent             | LangChain AI Agent              | Generates slug, title, and meta description   | Send a text message1     | Parsing                 |                                                   |
| Parsing              | Code                           | Parses AI output JSON string                    | AI Agent                 | Copywriter AI Agent     |                                                   |
| Copywriter AI Agent  | LangChain AI Agent              | Writes SEO optimized blog post HTML             | Parsing                  | Cleanup HTML            | ## Write SEO Optimized Blog Post                    |
| Cleanup HTML          | Set Node                       | Cleans AI-generated HTML content                | Copywriter AI Agent      | HTTP Request            |                                                   |
| HTTP Request          | HTTP Request                   | Publishes blog post to Blogger                  | Cleanup HTML             | Send a text message     |                                                   |
| Send a text message   | Telegram                      | Sends success confirmation via Telegram         | HTTP Request             |                         |                                                   |
| Sticky Note7          | Sticky Note                    | Label: "Create Title, Slug & Meta"             |                         |                         | ## Create Title, Slug & Meta                        |
| Sticky Note12         | Sticky Note (disabled)          | (Empty, disabled)                               |                         |                         |                                                   |
| Sticky Note6          | Sticky Note                    | Label: "Write SEO Optimized Blog Post"          |                         |                         | ## Write SEO Optimized Blog Post                    |
| Sticky Note8          | Sticky Note (disabled)          | Label: "Get NEWS" (disabled)                    |                         |                         |                                                   |
| OpenRouter Chat Model2 | OpenRouter Chat Model          | AI model used by Copywriter AI Agent            |                         | Copywriter AI Agent     |                                                   |
| OpenRouter Chat Model | OpenRouter Chat Model          | AI model used by AI Agent                        |                         | AI Agent                |                                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node**
   - Type: Schedule Trigger
   - Set to trigger every 1 minute.
   - Connect output to `Mediastack News`.

2. **Create an HTTP Request node named "Mediastack News"**
   - Method: GET
   - URL: `https://api.mediastack.com/v1/news?categories=technology&limit=1&languages=en`
   - Authentication: HTTP Query Auth with your Mediastack API key.
   - Connect output to `Genarate image`.

3. **Create an HTTP Request node named "Genarate image"**
   - Method: GET
   - URL: `https://api.pexels.com/v1/search`
   - Query Parameters:
     - `query`: Expression referencing news title from previous node.
     - `per_page`: 2
   - Authentication: HTTP Header Auth with Pexels API key.
   - Connect output to `Send a text message1`.

4. **Create a Telegram node named "Send a text message1"**
   - Action: Send Message
   - Chat ID: Use variable `$vars.telegramchatid`
   - Text: Compose message with landscape URLs of images from previous node.
   - Credentials: Telegram API credentials.
   - Connect output to `AI Agent`.

5. **Create a LangChain AI Agent node named "AI Agent"**
   - Prompt: Provide detailed instructions to generate slug, blog title, and meta description from the news title.
   - Use an OpenRouter Chat Model node as the AI language model.
   - Connect output to `Parsing`.

6. **Create a Code node named "Parsing"**
   - JavaScript code: Remove markdown JSON code block syntax from input and parse JSON string.
   - Connect output to `Copywriter AI Agent`.

7. **Create a LangChain AI Agent node named "Copywriter AI Agent"**
   - Prompt: Detailed instructions to write a 1000-word SEO blog post with embedded keywords, images, headings, and structured HTML.
   - Use OpenRouter Chat Model2 as AI language model.
   - Connect output to `Cleanup HTML`.

8. **Create a Set node named "Cleanup HTML"**
   - Assign field `content` by replacing all occurrences of "```html" and "```" from AI-generated content.
   - Connect output to `HTTP Request (Publish)`.

9. **Create an HTTP Request node named "HTTP Request"**
   - Method: POST
   - URL: `https://www.googleapis.com/blogger/v3/blogs/$vars.bloggerid/posts`
   - Body (JSON):
     ```json
     {
       "kind": "blogger#post",
       "blog": { "id": "{{$vars.bloggerid}}" },
       "title": "{{ $('Parsing').first().json.title }}",
       "content": "{{ $json.content.replaceAll('\n','').replaceAll('\"','\'').replaceAll('>        <','').replaceAll('>    <','').split('\n\n') }}"
     }
     ```
   - Authentication: OAuth2 (Google API)
   - Headers: Content-Type: application/json
   - Connect output to `Send a text message`.

10. **Create a Telegram node named "Send a text message"**
    - Action: Send Message
    - Chat ID: `$vars.telegramchatid`
    - Text: Confirmation message with current timestamp and blog post title.
    - Credentials: Telegram API credentials.

11. **Set up environment variables or workflow variables**
    - `$vars.telegramchatid`: Telegram chat ID to receive messages.
    - `$vars.bloggerid`: Blogger blog ID for publishing.
    - Credentials:
      - Mediastack API key (HTTP Query Auth)
      - Pexels API key (HTTP Header Auth)
      - Telegram API credentials (Bot token)
      - OpenRouter API credentials for AI nodes
      - Google OAuth2 credentials for Blogger API

12. **Add Sticky Notes for clarity (optional)**
    - Label blocks such as "Create Title, Slug & Meta" and "Write SEO Optimized Blog Post" for maintainability.

---

### 5. General Notes & Resources

| Note Content                                                                                                                          | Context or Link                                         |
|----------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------|
| The workflow generates SEO-optimized blog posts targeting recruitment/HR keywords, leveraging real-time technology news and AI writing. | Workflow purpose                                        |
| OpenRouter AI model "microsoft/mai-ds-r1:free" is used for AI text generation.                                                         | AI model reference                                     |
| Telegram integration requires setting the chat ID in `$vars.telegramchatid` variable.                                                   | Telegram messaging configuration                         |
| Blogger publishing requires Google OAuth2 credentials and setting blog ID in `$vars.bloggerid`.                                         | Blogger API integration                                 |
| Pexels API is used to retrieve relevant images based on news titles with a limit of two images.                                        | Image sourcing API                                     |
| Mediastack API limits news retrieval to one recent technology article per trigger.                                                     | News source limitation                                  |

---

_Disclaimer: The provided text originates exclusively from an automated workflow created with n8n, respecting applicable content policies and containing only legal and public data._