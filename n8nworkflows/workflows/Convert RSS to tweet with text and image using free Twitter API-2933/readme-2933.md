Convert RSS to tweet with text and image using free Twitter API

https://n8nworkflows.xyz/workflows/convert-rss-to-tweet-with-text-and-image-using-free-twitter-api-2933


# Convert RSS to tweet with text and image using free Twitter API

### 1. Workflow Overview

This workflow automates the process of converting new RSS feed articles into engaging tweets on X (formerly Twitter), including both text and images, using free Twitter API access. It targets content creators, marketers, and X users who want to grow their audience efficiently without manual posting.

The workflow is logically divided into the following blocks:

- **1.1 RSS Feed Trigger & Content Retrieval:** Detects new articles from an RSS feed and fetches their full HTML content.
- **1.2 Image Extraction:** Parses the article HTML to extract the main image URL.
- **1.3 AI-Powered Tweet Generation:** Uses AI models (OpenAI, Gemini, Perplexity, configurable) to generate a unique, engaging tweet text under 280 characters.
- **1.4 Tweet Validation & Image Handling:** Checks if the tweet meets length constraints; if an image is present, downloads and uploads it to Twitter’s server.
- **1.5 Tweet Posting:** Publishes the tweet with text and optional image via the free Twitter API.

---

### 2. Block-by-Block Analysis

#### 2.1 RSS Feed Trigger & Content Retrieval

- **Overview:**  
  This block listens for new articles in the configured RSS feed and retrieves the full HTML content of the latest article for further processing.

- **Nodes Involved:**  
  - Get the latest article from the feed  
  - Fetches the article’s HTML content

- **Node Details:**

  - **Get the latest article from the feed**  
    - Type: RSS Feed Read Trigger  
    - Role: Entry point; triggers workflow on new RSS feed items.  
    - Configuration: Uses default RSS feed URL (user-configurable).  
    - Inputs: None (trigger node)  
    - Outputs: Article metadata including link, title, and summary.  
    - Edge cases: Feed downtime, malformed RSS, no new items.

  - **Fetches the article’s HTML content**  
    - Type: HTTP Request  
    - Role: Retrieves full HTML content of the article URL from the RSS item.  
    - Configuration: HTTP GET request to article URL extracted from RSS item.  
    - Inputs: URL from previous node.  
    - Outputs: Raw HTML content of the article page.  
    - Edge cases: HTTP errors (404, 500), redirects, slow response, invalid URLs.

#### 2.2 Image Extraction

- **Overview:**  
  Parses the article’s HTML content to extract the main image URL for inclusion in the tweet.

- **Nodes Involved:**  
  - Extracts the main image

- **Node Details:**

  - **Extracts the main image**  
    - Type: HTML  
    - Role: Parses HTML to find the primary image (e.g., og:image meta tag or main article image).  
    - Configuration: Uses CSS selectors or XPath expressions to locate image URL.  
    - Inputs: HTML content from previous node.  
    - Outputs: Image URL string.  
    - Edge cases: Missing images, multiple images, malformed HTML.

#### 2.3 AI-Powered Tweet Generation

- **Overview:**  
  Generates a concise, engaging tweet text summarizing the article using AI language models.

- **Nodes Involved:**  
  - Summarize the Website API Perplexity  
  - OpenRouter Chat Model  
  - AI Agent

- **Node Details:**

  - **Summarize the Website API Perplexity**  
    - Type: HTTP Request  
    - Role: Calls Perplexity API to summarize the article content.  
    - Configuration: Sends article URL or content to Perplexity API endpoint.  
    - Inputs: Image extraction output or article content.  
    - Outputs: Summary text.  
    - Edge cases: API rate limits, network errors, invalid API keys.

  - **OpenRouter Chat Model**  
    - Type: Langchain OpenRouter Chat Model  
    - Role: Provides AI language model interface (configurable to OpenAI, Gemini, etc.).  
    - Configuration: API credentials and model parameters set in node.  
    - Inputs: Summarized text or prompts.  
    - Outputs: AI-generated tweet text.  
    - Edge cases: Authentication errors, model timeouts, malformed prompts.

  - **AI Agent**  
    - Type: Langchain Agent  
    - Role: Orchestrates AI model calls and processes outputs to generate final tweet text.  
    - Configuration: Uses OpenRouter Chat Model as language model.  
    - Inputs: Outputs from AI models and summary nodes.  
    - Outputs: Final tweet text.  
    - Edge cases: Expression evaluation errors, AI response failures.

#### 2.4 Tweet Validation & Image Handling

- **Overview:**  
  Validates tweet length constraints and manages image download and upload to Twitter’s media server.

- **Nodes Involved:**  
  - Verify tweet constraints  
  - Downloads image  
  - Upload image to X server with Twitter API v1

- **Node Details:**

  - **Verify tweet constraints**  
    - Type: If  
    - Role: Checks if generated tweet text fits within 280 characters (or bypasses for verified users).  
    - Configuration: Conditional expression on tweet text length.  
    - Inputs: AI Agent output.  
    - Outputs: Branches to either direct tweet or image download.  
    - Edge cases: Tweets exceeding limit, empty text.

  - **Downloads image**  
    - Type: HTTP Request  
    - Role: Downloads the extracted image from the URL for upload.  
    - Configuration: HTTP GET request with binary output enabled.  
    - Inputs: Image URL from extraction node.  
    - Outputs: Binary image data.  
    - Edge cases: Broken image URLs, large file sizes, timeouts.

  - **Upload image to X server with Twitter API v1**  
    - Type: HTTP Request  
    - Role: Uploads binary image data to Twitter’s media upload endpoint.  
    - Configuration: Uses Twitter API v1 credentials, multipart/form-data POST.  
    - Inputs: Binary image data from download node.  
    - Outputs: Media ID for tweet attachment.  
    - Edge cases: API authentication errors, upload failures, size limits.

#### 2.5 Tweet Posting

- **Overview:**  
  Posts the final tweet with text and optional image media ID to X using the free Twitter API.

- **Nodes Involved:**  
  - X

- **Node Details:**

  - **X**  
    - Type: Twitter  
    - Role: Publishes the tweet on the user’s X account.  
    - Configuration: Uses OAuth2 credentials for Twitter API v2, supports text and media ID parameters.  
    - Inputs: Tweet text from AI Agent and media ID from image upload node.  
    - Outputs: Tweet confirmation and metadata.  
    - Edge cases: API rate limits, authentication errors, tweet duplication errors.

---

### 3. Summary Table

| Node Name                               | Node Type                        | Functional Role                         | Input Node(s)                           | Output Node(s)                             | Sticky Note                      |
|----------------------------------------|---------------------------------|---------------------------------------|---------------------------------------|--------------------------------------------|---------------------------------|
| Get the latest article from the feed   | RSS Feed Read Trigger            | Trigger on new RSS article             | None                                  | Fetches the article’s HTML content          |                                 |
| Fetches the article’s HTML content     | HTTP Request                    | Retrieves full article HTML            | Get the latest article from the feed  | Extracts the main image                      |                                 |
| Extracts the main image                 | HTML                            | Parses HTML to extract main image URL | Fetches the article’s HTML content    | Summarize the Website API Perplexity        |                                 |
| Summarize the Website API Perplexity   | HTTP Request                    | Summarizes article content via API    | Extracts the main image                | AI Agent                                    |                                 |
| OpenRouter Chat Model                   | Langchain OpenRouter Chat Model | Provides AI language model             | None (used by AI Agent)                | AI Agent                                    |                                 |
| AI Agent                              | Langchain Agent                 | Generates final tweet text             | Summarize the Website API Perplexity, Verify tweet constraints | Verify tweet constraints                   |                                 |
| Verify tweet constraints                | If                             | Checks tweet length constraints        | AI Agent                              | AI Agent (valid), Downloads image (invalid) |                                 |
| Downloads image                        | HTTP Request                    | Downloads image binary data            | Verify tweet constraints              | Upload image to X server with Twitter API v1 |                                 |
| Upload image to X server with Twitter API v1 | HTTP Request                    | Uploads image to Twitter media server | Downloads image                      | X                                          |                                 |
| X                                     | Twitter                         | Posts tweet with text and image        | Upload image to X server with Twitter API v1, AI Agent | None                                       |                                 |
| Sticky Note1                          | Sticky Note                    | (No content)                          | None                                  | None                                       |                                 |
| Sticky Note2                          | Sticky Note                    | (No content)                          | None                                  | None                                       |                                 |
| Sticky Note3                          | Sticky Note                    | (No content)                          | None                                  | None                                       |                                 |
| Sticky Note                           | Sticky Note                    | (No content)                          | None                                  | None                                       |                                 |
| Sticky Note4                          | Sticky Note                    | (No content)                          | None                                  | None                                       |                                 |
| Sticky Note5                          | Sticky Note                    | (No content)                          | None                                  | None                                       |                                 |
| Sticky Note6                          | Sticky Note                    | (No content)                          | None                                  | None                                       |                                 |
| Sticky Note7                          | Sticky Note                    | (No content)                          | None                                  | None                                       |                                 |
| Sticky Note8                          | Sticky Note                    | (No content)                          | None                                  | None                                       |                                 |
| Sticky Note9                          | Sticky Note                    | (No content)                          | None                                  | None                                       |                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create RSS Feed Read Trigger node**  
   - Name: "Get the latest article from the feed"  
   - Type: RSS Feed Read Trigger  
   - Configuration: Set RSS feed URL for your content source.  
   - Purpose: Trigger workflow on new RSS items.

2. **Add HTTP Request node**  
   - Name: "Fetches the article’s HTML content"  
   - Type: HTTP Request  
   - Configuration:  
     - HTTP Method: GET  
     - URL: Use expression to get article link from RSS trigger (e.g., `{{$json["link"]}}`)  
   - Connect from RSS Feed Trigger node.

3. **Add HTML node**  
   - Name: "Extracts the main image"  
   - Type: HTML  
   - Configuration:  
     - Use CSS selector or XPath to extract main image URL, e.g., meta property `og:image` or first `<img>` in article content.  
   - Connect from HTTP Request node.

4. **Add HTTP Request node for Perplexity API**  
   - Name: "Summarize the Website API Perplexity"  
   - Type: HTTP Request  
   - Configuration:  
     - Method: POST or GET depending on API  
     - URL: Perplexity API endpoint  
     - Body: Pass article URL or extracted content  
     - Authentication: Set API key if required  
   - Connect from HTML node.

5. **Add Langchain OpenRouter Chat Model node**  
   - Name: "OpenRouter Chat Model"  
   - Type: Langchain OpenRouter Chat Model  
   - Configuration:  
     - Set API credentials for OpenRouter or preferred AI provider  
     - Configure model parameters (temperature, max tokens, etc.)  
   - No direct input; used by AI Agent.

6. **Add Langchain Agent node**  
   - Name: "AI Agent"  
   - Type: Langchain Agent  
   - Configuration:  
     - Set language model to "OpenRouter Chat Model" node  
     - Define prompt template to generate tweet text under 280 characters  
   - Connect from "Summarize the Website API Perplexity" node.

7. **Add If node**  
   - Name: "Verify tweet constraints"  
   - Type: If  
   - Configuration:  
     - Condition: Check if tweet text length ≤ 280 characters (or bypass for verified users)  
     - Expression example: `{{$json["tweetText"].length <= 280}}`  
   - Connect from AI Agent node.

8. **Add HTTP Request node to download image**  
   - Name: "Downloads image"  
   - Type: HTTP Request  
   - Configuration:  
     - Method: GET  
     - URL: Use extracted image URL from "Extracts the main image" node  
     - Response Format: File / Binary  
   - Connect from "Verify tweet constraints" node (false branch).

9. **Add HTTP Request node to upload image to Twitter**  
   - Name: "Upload image to X server with Twitter API v1"  
   - Type: HTTP Request  
   - Configuration:  
     - Method: POST  
     - URL: Twitter API v1 media upload endpoint  
     - Authentication: Twitter OAuth1 or OAuth2 credentials  
     - Body: Multipart/form-data with binary image data  
   - Connect from "Downloads image" node.

10. **Add Twitter node**  
    - Name: "X"  
    - Type: Twitter  
    - Configuration:  
      - OAuth2 credentials for Twitter API  
      - Parameters: Tweet text from AI Agent, media ID from image upload node (if available)  
    - Connect from "Upload image to X server with Twitter API v1" node (image branch) and from "Verify tweet constraints" node (true branch).

11. **Set up credentials**  
    - Twitter OAuth2 or OAuth1 credentials with appropriate permissions for posting tweets and uploading media.  
    - OpenAI or other AI provider credentials for Langchain nodes.  
    - Perplexity API key if required.

12. **Test the workflow**  
    - Trigger with a new RSS feed item.  
    - Verify tweet text generation, image extraction, upload, and posting.

---

### 5. General Notes & Resources

| Note Content                                                                                      | Context or Link                                                                                   |
|-------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| The workflow includes a step-by-step video tutorial for easy setup and customization.            | Included in the downloaded file with the workflow.                                              |
| This automation uses free Twitter API access, avoiding extra subscription costs for posting.     | Workflow description.                                                                            |
| AI models are configurable; users can switch between OpenAI, Gemini, Perplexity, or others.      | Workflow description and Langchain node configurations.                                         |
| Ideal for content creators and marketers aiming to automate social media growth on X.            | Workflow description.                                                                            |

---

This document provides a comprehensive understanding of the workflow structure, node configurations, and reproduction steps to enable efficient use, modification, and troubleshooting.