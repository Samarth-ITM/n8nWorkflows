Generate and Publish SEO-Optimized Blog Posts to Blogger with OpenAI & DALL-E

https://n8nworkflows.xyz/workflows/generate-and-publish-seo-optimized-blog-posts-to-blogger-with-openai---dall-e-4905


# Generate and Publish SEO-Optimized Blog Posts to Blogger with OpenAI & DALL-E

### 1. Workflow Overview

This workflow automates the generation and publication of SEO-optimized blog posts on Blogger, leveraging OpenAI language models and DALL·E for content and image creation. It is designed for content creators or marketers who want to streamline blog production from receiving a blog title via Telegram to publishing a rich, illustrated article and notifying the user.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Receives the blog post title from a Telegram trigger and sends a confirmation notification.
- **1.2 Blogger Profile Retrieval:** Fetches Blogger profile details required for post publication.
- **1.3 Metadata Generation:** Uses OpenAI models to generate structured metadata (e.g., SEO keywords, summary) for the blog post.
- **1.4 Full Blog Content Writing:** Generates the complete blog content based on the metadata.
- **1.5 Custom URL Creation:** Creates a SEO-friendly custom URL for the blog post.
- **1.6 Image Generation and Upload:** Generates a blog post image using DALL·E, uploads it to Imgur, and embeds it into the blog content.
- **1.7 Publishing and Notification:** Publishes the blog post to Blogger and sends the published post link back to the user on Telegram.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  Captures the incoming blog title from a Telegram message and notifies the user that the title has been received.

- **Nodes Involved:**  
  - `🟢 Start: Telegram Input`  
  - `📨 Notify: Title Received`

- **Node Details:**

  1. **🟢 Start: Telegram Input**  
     - Type: Telegram Trigger  
     - Role: Listens for incoming Telegram messages to start the workflow  
     - Configuration: Uses a webhook ID for Telegram integration; triggers when a user sends a message  
     - Inputs: External Telegram message  
     - Outputs: Passes message data (blog title) to the next node  
     - Edge Cases: Telegram API authentication errors, message format issues, webhook downtime  
     - Sub-workflow: None

  2. **📨 Notify: Title Received**  
     - Type: Telegram  
     - Role: Sends a Telegram message back to the user confirming title reception  
     - Configuration: Uses Telegram credentials and webhook ID to send message  
     - Inputs: Receives blog title from the trigger node  
     - Outputs: Triggers Blogger profile retrieval  
     - Edge Cases: Message send failure due to Telegram rate limits, invalid chat ID

#### 2.2 Blogger Profile Retrieval

- **Overview:**  
  Retrieves Blogger profile information necessary for API calls and blog post publishing.

- **Nodes Involved:**  
  - `🔑 Get Blogger Profile`

- **Node Details:**

  1. **🔑 Get Blogger Profile**  
     - Type: HTTP Request  
     - Role: Calls Blogger API to fetch user profile and blog IDs  
     - Configuration: Authenticated request with OAuth2 credentials for Blogger; GET request to Blogger user/profile endpoint  
     - Inputs: Triggered after notification node  
     - Outputs: Passes Blogger profile info to metadata generation  
     - Edge Cases: OAuth token expiry, API rate limits, network failures

#### 2.3 Metadata Generation

- **Overview:**  
  Uses OpenAI chat models to generate structured metadata for the blog post such as the SEO title, keywords, and summary.

- **Nodes Involved:**  
  - `🧠 AI Model: Metadata Generator`  
  - `📄 Extract Metadata Structure`  
  - `🧠 Generate Blog Metadata`

- **Node Details:**

  1. **🧠 AI Model: Metadata Generator**  
     - Type: LangChain OpenAI Chat Model  
     - Role: Produces raw metadata text from the blog title and profile data  
     - Configuration: OpenAI Chat model with prompt tailored to generate metadata  
     - Inputs: Blogger profile and blog title  
     - Outputs: Raw metadata text to output parser  
     - Edge Cases: API quota exceeded, prompt generation errors, malformed input

  2. **📄 Extract Metadata Structure**  
     - Type: LangChain Output Parser Structured  
     - Role: Parses raw AI output into structured JSON metadata  
     - Configuration: Schema for metadata fields (e.g., SEO keywords, summary)  
     - Inputs: Raw metadata text from AI model  
     - Outputs: Structured metadata JSON to metadata chain LLM  
     - Edge Cases: Parsing errors if AI output deviates from schema

  3. **🧠 Generate Blog Metadata**  
     - Type: LangChain Chain LLM  
     - Role: Coordinates metadata generation and parsing steps  
     - Configuration: Combines above two nodes logically  
     - Inputs: Blogger profile data  
     - Outputs: Passes structured metadata to blog content writer  
     - Edge Cases: Workflow retry enabled on failure to handle transient AI errors

#### 2.4 Full Blog Content Writing

- **Overview:**  
  Generates the full blog post content based on the generated metadata.

- **Nodes Involved:**  
  - `🧠 AI Model: Article Writer`  
  - `✍️ Write Full Blog Content`

- **Node Details:**

  1. **🧠 AI Model: Article Writer**  
     - Type: LangChain OpenAI Chat Model  
     - Role: Uses metadata to generate full blog article text  
     - Configuration: OpenAI Chat model with article writing prompt  
     - Inputs: Metadata JSON  
     - Outputs: Blog content text to chain LLM  
     - Edge Cases: API limits, content generation errors

  2. **✍️ Write Full Blog Content**  
     - Type: LangChain Chain LLM  
     - Role: Orchestrates article writing flow including retries  
     - Configuration: Retry on fail enabled  
     - Inputs: Metadata from metadata generation block  
     - Outputs: Blog content text for URL creation  
     - Edge Cases: Retry handles transient failures; prolonged failures may halt workflow

#### 2.5 Custom URL Creation

- **Overview:**  
  Generates a SEO-friendly custom URL slug for the blog post.

- **Nodes Involved:**  
  - `🔗 Create Custom Blog Post URL`

- **Node Details:**

  1. **🔗 Create Custom Blog Post URL**  
     - Type: Code Node  
     - Role: Creates URL slug from blog title or metadata  
     - Configuration: Custom JavaScript code to slugify title (e.g., lowercase, hyphens, remove special chars)  
     - Inputs: Blog content or title data  
     - Outputs: URL slug for image generation  
     - Edge Cases: Title with unsupported characters, empty title

#### 2.6 Image Generation and Upload

- **Overview:**  
  Generates a blog image with DALL·E via OpenAI, uploads it to Imgur, and prepares it for embedding into the blog post.

- **Nodes Involved:**  
  - `🖼️ Generate Blog Image`  
  - `☁️ Upload Image to Imgur`  
  - `🧷 Embed Image in HTML`

- **Node Details:**

  1. **🖼️ Generate Blog Image**  
     - Type: LangChain OpenAI Node (DALL·E)  
     - Role: Creates an AI-generated image based on blog content or URL slug  
     - Configuration: Uses OpenAI DALL·E API; parameters include prompt and image size  
     - Inputs: URL slug from previous node  
     - Outputs: Image data (base64 or URL) to HTTP request node  
     - Edge Cases: API quota, prompt failures, image generation delays

  2. **☁️ Upload Image to Imgur**  
     - Type: HTTP Request  
     - Role: Uploads the generated image to Imgur for hosting and sharing  
     - Configuration: POST request to Imgur API with image payload; uses Imgur OAuth2 or client ID  
     - Inputs: Image data from DALL·E node  
     - Outputs: Imgur image URL for embedding  
     - Edge Cases: Imgur rate limits, upload errors, invalid image data

  3. **🧷 Embed Image in HTML**  
     - Type: Set Node  
     - Role: Embeds Imgur image URL into the blog post HTML content  
     - Configuration: Sets HTML content field with `<img>` tag including Imgur URL  
     - Inputs: Imgur image URL and blog content  
     - Outputs: Final blog HTML content to publishing node  
     - Edge Cases: Missing image URL, malformed HTML

#### 2.7 Publishing and Notification

- **Overview:**  
  Publishes the finalized blog post to Blogger and sends a confirmation message with the link back to the user on Telegram.

- **Nodes Involved:**  
  - `🚀 Publish Article to Blogger`  
  - `📬 Send Blog Link to User`

- **Node Details:**

  1. **🚀 Publish Article to Blogger**  
     - Type: HTTP Request  
     - Role: Posts the blog content with metadata and image to Blogger API to publish the post  
     - Configuration: POST request with OAuth2 credentials; includes blog ID, post title, content, custom URL slug, and metadata  
     - Inputs: Final HTML content with embedded image and URL slug  
     - Outputs: Blogger post URL to Telegram notification  
     - Edge Cases: OAuth token expiry, API limits, content validation errors

  2. **📬 Send Blog Link to User**  
     - Type: Telegram  
     - Role: Sends the published blog post link back to the user on Telegram  
     - Configuration: Uses Telegram webhook ID and credentials to send message  
     - Inputs: Blogger post URL  
     - Outputs: End of workflow  
     - Edge Cases: Telegram send failures, invalid chat ID

---

### 3. Summary Table

| Node Name                   | Node Type                                   | Functional Role                   | Input Node(s)                  | Output Node(s)                 | Sticky Note                     |
|-----------------------------|---------------------------------------------|---------------------------------|-------------------------------|-------------------------------|--------------------------------|
| 🟢 Start: Telegram Input      | Telegram Trigger                            | Receive blog title input         | —                             | 📨 Notify: Title Received      |                                |
| 📨 Notify: Title Received     | Telegram                                   | Notify user title received       | 🟢 Start: Telegram Input       | 🔑 Get Blogger Profile         |                                |
| 🔑 Get Blogger Profile        | HTTP Request                               | Fetch Blogger profile info       | 📨 Notify: Title Received      | 🧠 Generate Blog Metadata      |                                |
| 🧠 AI Model: Metadata Generator | LangChain OpenAI Chat Model                 | Generate raw metadata            | 🔑 Get Blogger Profile         | 📄 Extract Metadata Structure  |                                |
| 📄 Extract Metadata Structure | LangChain Output Parser Structured         | Parse metadata JSON              | 🧠 AI Model: Metadata Generator | 🧠 Generate Blog Metadata      |                                |
| 🧠 Generate Blog Metadata     | LangChain Chain LLM                        | Metadata generation orchestration | 📄 Extract Metadata Structure  | ✍️ Write Full Blog Content     |                                |
| ✍️ Write Full Blog Content     | LangChain Chain LLM                        | Generate full blog content       | 🧠 Generate Blog Metadata      | 🔗 Create Custom Blog Post URL |                                |
| 🔗 Create Custom Blog Post URL | Code                                       | Create SEO-friendly URL slug     | ✍️ Write Full Blog Content     | 🖼️ Generate Blog Image         |                                |
| 🖼️ Generate Blog Image         | LangChain OpenAI (DALL·E)                  | Generate blog image              | 🔗 Create Custom Blog Post URL | ☁️ Upload Image to Imgur       |                                |
| ☁️ Upload Image to Imgur       | HTTP Request                               | Upload image to Imgur            | 🖼️ Generate Blog Image         | 🧷 Embed Image in HTML         |                                |
| 🧷 Embed Image in HTML         | Set                                         | Embed image URL in blog HTML    | ☁️ Upload Image to Imgur       | 🚀 Publish Article to Blogger  |                                |
| 🚀 Publish Article to Blogger  | HTTP Request                               | Publish blog post to Blogger    | 🧷 Embed Image in HTML         | 📬 Send Blog Link to User      |                                |
| 📬 Send Blog Link to User      | Telegram                                   | Notify user with blog URL       | 🚀 Publish Article to Blogger  | —                             |                                |
| Sticky Note                  | Sticky Note                                | N/A                             | N/A                           | N/A                           |                                |
| Sticky Note1                 | Sticky Note                                | N/A                             | N/A                           | N/A                           |                                |
| Sticky Note2                 | Sticky Note                                | N/A                             | N/A                           | N/A                           |                                |
| Sticky Note3                 | Sticky Note                                | N/A                             | N/A                           | N/A                           |                                |
| Sticky Note4                 | Sticky Note                                | N/A                             | N/A                           | N/A                           |                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger Node**  
   - Type: Telegram Trigger  
   - Configure with Telegram Bot credentials and set webhook ID  
   - Trigger on incoming messages to receive blog post titles

2. **Create Telegram Node to Notify Title Reception**  
   - Type: Telegram  
   - Configure with same Telegram credentials  
   - Send a confirmation message to the user acknowledging title receipt  
   - Connect output of Telegram Trigger to this node

3. **Create HTTP Request Node to Get Blogger Profile**  
   - Type: HTTP Request  
   - Configure OAuth2 credentials for Blogger API  
   - Set method to GET for Blogger user profile endpoint (`https://www.googleapis.com/blogger/v3/users/self`)  
   - Connect output of Telegram notification node here

4. **Create LangChain OpenAI Chat Model Node for Metadata Generation**  
   - Type: LangChain chainLlm or lmChatOpenAi  
   - Configure with OpenAI API credentials  
   - Set prompt to generate SEO metadata (keywords, summary) based on blog title and Blogger profile  
   - Connect output of Blogger profile HTTP node here

5. **Create LangChain Output Parser Node**  
   - Type: OutputParserStructured  
   - Define schema for metadata fields (e.g., title, keywords, summary)  
   - Connect output of AI Metadata Generator node here

6. **Create LangChain Chain LLM Node to Orchestrate Metadata Generation**  
   - Type: Chain LLM  
   - Combine AI model and parser nodes logically  
   - Enable retry on failure to handle AI API errors  
   - Connect output of Output Parser node here

7. **Create LangChain OpenAI Chat Model Node for Blog Content Writing**  
   - Type: LangChain chainLlm or lmChatOpenAi  
   - Use metadata as input to generate full blog content  
   - Connect output of metadata generation chain here

8. **Create Chain LLM Node for Writing Full Blog Content**  
   - Type: Chain LLM  
   - Enable retry on fail for stability  
   - Connect output of AI Article Writer node here

9. **Create Code Node to Generate Custom SEO-friendly URL**  
   - Type: Code  
   - Write JavaScript to slugify blog title or metadata (lowercase, hyphens, remove special chars)  
   - Connect output of blog content writer node here

10. **Create LangChain OpenAI Node for Image Generation**  
    - Type: LangChain OpenAI (DALL·E)  
    - Use custom URL slug to create image prompt  
    - Configure OpenAI DALL·E credentials and parameters (size, style)  
    - Connect output of URL creation node here

11. **Create HTTP Request Node to Upload Image to Imgur**  
    - Type: HTTP Request  
    - Configure with Imgur API credentials (OAuth2 or client ID)  
    - Set POST method to Imgur image upload endpoint  
    - Pass image data from DALL·E node  
    - Connect output of image generation node here

12. **Create Set Node to Embed Image in Blog HTML Content**  
    - Type: Set  
    - Use Imgur image URL to create an `<img>` tag embedded in blog content HTML  
    - Connect output of Imgur upload node here

13. **Create HTTP Request Node to Publish Blog Post to Blogger**  
    - Type: HTTP Request  
    - Configure with Blogger OAuth2 credentials  
    - Set POST method to Blogger posts endpoint (`https://www.googleapis.com/blogger/v3/blogs/{blogId}/posts/`)  
    - Include blog title, content (with embedded image), custom URL slug, and metadata in request body  
    - Connect output of Set node here

14. **Create Telegram Node to Send Published Blog URL to User**  
    - Type: Telegram  
    - Configure with Telegram credentials  
    - Send a message containing the Blogger post URL to the user  
    - Connect output of Blogger publish node here

15. **Test Workflow**  
    - Verify Telegram message triggers workflow correctly  
    - Confirm metadata and content generation is accurate  
    - Ensure image is generated, uploaded, and embedded properly  
    - Validate Blogger post publishes successfully  
    - Confirm user receives the post link on Telegram

---

### 5. General Notes & Resources

| Note Content                                                                                                 | Context or Link                                      |
|--------------------------------------------------------------------------------------------------------------|-----------------------------------------------------|
| This workflow relies on the integration of OpenAI APIs for both text and image generation (ChatGPT & DALL·E). | Requires valid OpenAI API key and usage quota       |
| Blogger API interaction requires OAuth2 credentials with appropriate scopes for reading profiles and publishing posts. | https://developers.google.com/blogger/docs/3.0/using |
| Imgur API is used for image hosting; ensure Imgur API credentials are configured correctly.                   | https://apidocs.imgur.com/                            |
| Telegram Bot setup requires webhook configuration and correct chat permissions for sending messages.         | https://core.telegram.org/bots/api                   |
| Use retry options in LangChain Chain LLM nodes to improve workflow robustness against transient API failures.| n8n LangChain node documentation                      |

---

**Disclaimer:** The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled are legal and public.