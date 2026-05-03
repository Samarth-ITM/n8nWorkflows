Research & Generate Blog Posts from Keywords with Dumpling AI and GPT-4 to WordPress

https://n8nworkflows.xyz/workflows/research---generate-blog-posts-from-keywords-with-dumpling-ai-and-gpt-4-to-wordpress-9250


# Research & Generate Blog Posts from Keywords with Dumpling AI and GPT-4 to WordPress

### 1. Workflow Overview

This workflow automates the generation and publication of SEO-optimized blog posts from a single input keyword. It is designed for content creators, SEO writers, and niche bloggers who want to leverage AI-driven research and writing tools to streamline their content production.

The logical flow is divided into these blocks:

- **1.1 Input Reception:** Captures a keyword input submitted via a web form.
- **1.2 Research and Data Extraction:** Uses Dumpling AI to perform a Google search and extracts relevant search results, People Also Ask (PAA) questions, and related searches.
- **1.3 Content Generation:** Utilizes OpenAI’s GPT-4 to select the most insightful PAA question and generate a detailed blog post based on the keyword and search data.
- **1.4 Review Process:** Sends the generated blog post via Gmail for manual review and approval.
- **1.5 Publishing:** Upon approval, automatically publishes the blog post to a WordPress site.

This structure ensures a seamless flow from keyword input through AI-driven research and writing, review, and final publication.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  Receives the blog topic keyword via a form submission webhook, triggering the workflow.

- **Nodes Involved:**  
  - Trigger: Receive Keyword from Form

- **Node Details:**  

  - **Trigger: Receive Keyword from Form**  
    - Type: `Form Trigger`  
    - Role: Starts the workflow when a user submits a keyword via a form titled "ideas".  
    - Configuration: The form contains a single field labeled "search key".  
    - Input/Output: Outputs the submitted keyword as JSON under `search key`.  
    - Notes: Webhook URL generated for form integration.  
    - Potential Failures:  
      - Form submission errors or missing keyword input.  
      - Webhook connectivity issues.  

#### 2.2 Research and Data Extraction

- **Overview:**  
  Sends the received keyword to Dumpling AI to perform a Google search and extracts structured data from the search results including top organic results, People Also Ask questions, and related searches.

- **Nodes Involved:**  
  - Search Google via Dumpling AI  
  - Extract Top Results, PAA & Related Searches  
  - Check if People Also Ask Exists

- **Node Details:**  

  - **Search Google via Dumpling AI**  
    - Type: `HTTP Request`  
    - Role: Queries Dumpling AI’s API to get Google search results for the keyword.  
    - Configuration:  
      - POST request to `https://app.dumplingai.com/api/v1/search`.  
      - Body parameters include the keyword (`query`) and `scrapeResults` set to true to get detailed scraping data.  
      - Authorization via HTTP Header with Dumpling AI credentials.  
    - Input: Receives the keyword from the form trigger.  
    - Output: JSON search results containing organic listings, People Also Ask, and related searches data.  
    - Potential Failures:  
      - API authentication failure.  
      - Dumpling AI service downtime or rate limiting.  
      - Network timeouts.  

  - **Extract Top Results, PAA & Related Searches**  
    - Type: `Code (JavaScript)`  
    - Role: Parses the Dumpling AI search results to extract:  
      - Top two organic search results with metadata (title, URL, snippet, date, sitelinks, scraping metadata).  
      - People Also Ask questions and answers (if present).  
      - Top three related search queries.  
    - Key Expressions: Uses JavaScript to navigate nested JSON and safely extract data, returning a structured JSON object.  
    - Input: JSON from Dumpling AI node.  
    - Output: Structured object summarizing the key search insights.  
    - Potential Failures:  
      - Unexpected JSON structure causing extraction errors.  
      - Empty or missing fields in search results.  

  - **Check if People Also Ask Exists**  
    - Type: `Filter`  
    - Role: Checks if the extracted data contains any People Also Ask questions.  
    - Configuration: Filters on the field `peopleAlsoAskExists`, allowing the workflow to branch depending on its value ("Yes" or otherwise).  
    - Input: Output from the extraction code node.  
    - Output: Passes data forward only if PAA questions exist.  
    - Potential Failures:  
      - Logic errors if the field is missing or malformed.  

#### 2.3 Content Generation

- **Overview:**  
  Uses GPT-4 to analyze the search data and generate a detailed, original blog post focused around a selected People Also Ask question.

- **Nodes Involved:**  
  - Generate Blog Post with GPT-4

- **Node Details:**  

  - **Generate Blog Post with GPT-4**  
    - Type: `OpenAI (Langchain node)`  
    - Role: Generates blog content using GPT-4 chat completion model.  
    - Configuration:  
      - Model: GPT-4o latest.  
      - System prompt: Instructs the model to analyze the search key, top two results, and PAA questions to select one insightful PAA question and write a structured blog post.  
      - Message includes dynamic data expressions referencing the filtered extraction outputs.  
      - JSON output enforced with a strict format: `{ "title": string, "blog_post": string }`.  
    - Input: Data from the PAA filter node, with search key and related content.  
    - Output: JSON containing the blog post title and body.  
    - Credentials: OpenAI API key configured.  
    - Potential Failures:  
      - API quota or authentication issues.  
      - GPT-4 timeout or rate limits.  
      - Unexpected or malformed input causing poor generation results or JSON parse errors.  

#### 2.4 Review Process

- **Overview:**  
  Sends the generated blog post by email for manual review, expecting a double approval before publishing.

- **Nodes Involved:**  
  - Send Blog Post for Review via Gmail  
  - Check if Approved

- **Node Details:**  

  - **Send Blog Post for Review via Gmail**  
    - Type: `Gmail`  
    - Role: Sends an HTML-formatted email containing the blog title and body for human review.  
    - Configuration:  
      - Recipient email fixed as `example@gmail.com` (replace as needed).  
      - Email subject: "Workflow review needed".  
      - Email body: Styled HTML table showing blog title and content with placeholders dynamically filled from GPT-4 output.  
      - Uses Gmail OAuth2 credentials.  
      - Sends "sendAndWait" with double approval required, pausing workflow until approval is given.  
    - Input: Blog post JSON from GPT-4 node.  
    - Output: Approval status JSON after review.  
    - Potential Failures:  
      - Gmail OAuth token expiry or permission errors.  
      - Email sending failures or delays.  
      - Approval response missing or invalid.  

  - **Check if Approved**  
    - Type: `If`  
    - Role: Branches the workflow based on approval status.  
    - Condition: Checks if `data.approved` boolean equals `true`.  
    - Input: Approval data from Gmail node.  
    - Output:  
      - If approved: proceeds to publish.  
      - If not approved: loops back or halts (here it reconnects to GPT-4 to regenerate).  
    - Potential Failures:  
      - Missing or malformed approval data.  
      - Logic misrouting if condition fails unexpectedly.  

#### 2.5 Publishing

- **Overview:**  
  Publishes the approved blog post content directly to WordPress.

- **Nodes Involved:**  
  - Publish Blog Post to WordPress

- **Node Details:**  

  - **Publish Blog Post to WordPress**  
    - Type: `WordPress`  
    - Role: Creates a new blog post on a WordPress site using the generated title and content.  
    - Configuration:  
      - Post title and content dynamically assigned from GPT-4 generated data.  
      - Uses WordPress API credentials.  
    - Input: Blog post data after approval.  
    - Output: WordPress post creation response.  
    - Potential Failures:  
      - WordPress API authentication or permission errors.  
      - Content length or format issues causing post failure.  
      - Network or server errors.  

---

### 3. Summary Table

| Node Name                           | Node Type                      | Functional Role                            | Input Node(s)                  | Output Node(s)                   | Sticky Note                                                                                                                                                                                                                                                                                                |
|-----------------------------------|--------------------------------|-------------------------------------------|-------------------------------|---------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Trigger: Receive Keyword from Form | Form Trigger                   | Start workflow on keyword submission      | —                             | Search Google via Dumpling AI    | ## 📝 Generate Blog Post from Keyword Using Dumpling AI + GPT-4 This workflow creates and publishes a blog post from a single keyword, using AI for research, writing, and publishing.                                                                                                                     |
| Search Google via Dumpling AI      | HTTP Request                  | Query Dumpling AI API for Google results  | Trigger: Receive Keyword       | Extract Top Results, PAA & Related Searches | See above                                                                                                                                                                                                                                                                                                  |
| Extract Top Results, PAA & Related Searches | Code (JavaScript)            | Extracts top results, PAA questions, related searches | Search Google via Dumpling AI | Check if People Also Ask Exists  | See above                                                                                                                                                                                                                                                                                                  |
| Check if People Also Ask Exists    | Filter                        | Branch if PAA questions exist              | Extract Top Results            | Generate Blog Post with GPT-4    | See above                                                                                                                                                                                                                                                                                                  |
| Generate Blog Post with GPT-4      | OpenAI (Langchain)            | Generate blog post content from PAA data  | Check if People Also Ask Exists | Send Blog Post for Review via Gmail | See above                                                                                                                                                                                                                                                                                                  |
| Send Blog Post for Review via Gmail | Gmail                         | Send generated post for manual review     | Generate Blog Post with GPT-4  | Check if Approved                | See above                                                                                                                                                                                                                                                                                                  |
| Check if Approved                  | If                           | Branch on review approval status           | Send Blog Post for Review via Gmail | Publish Blog Post to WordPress / Generate Blog Post with GPT-4 | See above                                                                                                                                                                                                                                                                                                  |
| Publish Blog Post to WordPress     | WordPress                    | Publish approved blog post                  | Check if Approved             | —                               | See above                                                                                                                                                                                                                                                                                                  |
| Sticky Note                       | Sticky Note                   | Documentation and overview note             | —                             | —                               | ## 📝 Generate Blog Post from Keyword Using Dumpling AI + GPT-4 This workflow creates and publishes a blog post from a single keyword, using AI for research, writing, and publishing.                                                                                                                     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Form Trigger node**  
   - Type: Form Trigger  
   - Form title: `ideas`  
   - Add one form field labeled `search key`  
   - Save and note the generated webhook URL for form submissions.

2. **Add HTTP Request node: Search Google via Dumpling AI**  
   - Connect it to the Form Trigger node.  
   - Method: POST  
   - URL: `https://app.dumplingai.com/api/v1/search`  
   - Body parameters:  
     - `query`: expression `{{ $json['search key'] }}`  
     - `scrapeResults`: `true`  
   - Authentication: HTTP Header Auth with Dumpling AI API key credentials (create credential with your Dumpling AI token).  
   - Headers: none explicitly set beyond auth.

3. **Add Code node: Extract Top Results, PAA & Related Searches**  
   - Connect it to the HTTP Request node.  
   - Paste the provided JavaScript code that extracts:  
     - Top 2 organic results with metadata  
     - People Also Ask questions and answers  
     - Top 3 related searches  
   - Ensure no syntax errors and that it returns a structured JSON object.

4. **Add Filter node: Check if People Also Ask Exists**  
   - Connect it to the Code node.  
   - Condition: `$json.peopleAlsoAskExists` equals `Yes` (case-sensitive).  
   - This node filters flows to proceed only if PAA questions are present.

5. **Add OpenAI node (Langchain): Generate Blog Post with GPT-4**  
   - Connect it to the Filter node’s true output.  
   - Model: `chatgpt-4o-latest`  
   - Messages:  
     - System prompt instructing the AI to analyze search key, top results, and PAA to select a question and write a blog post.  
     - User prompt supplying dynamic data expressions referencing:  
       - Search key (`{{ $('Check if People Also Ask Exists').item.json.query }}`)  
       - Top two results titles and descriptions  
       - People Also Ask questions array  
   - Output: JSON parse enabled with expected schema `{title, blog_post}`.  
   - Credentials: Set with OpenAI API key.

6. **Add Gmail node: Send Blog Post for Review via Gmail**  
   - Connect it to the OpenAI node.  
   - Operation: `sendAndWait` with double approval required.  
   - Recipient: `example@gmail.com` (replace with actual reviewer email).  
   - Subject: "Workflow review needed"  
   - Email body: Use HTML format with placeholders:  
     - Title: `{{ $json.message.content.title }}`  
     - Blog post body: `{{ $json.message.content.blog_post }}`  
   - Credentials: Configure Gmail OAuth2 credentials.

7. **Add If node: Check if Approved**  
   - Connect it to the Gmail node.  
   - Condition: `$json.data.approved` equals `true`.  
   - True branch: proceeds to WordPress publishing.  
   - False branch: loops back to GPT-4 node to regenerate content.

8. **Add WordPress node: Publish Blog Post to WordPress**  
   - Connect it to the If node’s true output.  
   - Configure:  
     - Title: `{{ $('Generate Blog Post with GPT-4').item.json.message.content.title }}`  
     - Content: `{{ $('Generate Blog Post with GPT-4').item.json.message.content.blog_post }}`  
   - Credentials: Set up WordPress API credentials with permissions to create posts.

9. **Add a Sticky Note node** (optional)  
   - Add documentation or instructions for users or administrators.

10. **Workflow Settings:**  
    - Execution order: Default (v1) is suitable.  
    - Activate the workflow once all nodes are tested.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                        | Context or Link                                                                                                            |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------|
| This workflow requires active API credentials for Dumpling AI, OpenAI (GPT-4), Gmail (OAuth2), and WordPress (API with post publishing rights). Ensure all tokens and permissions are valid to avoid runtime errors.                                                                                                                                                              | Credential setup in n8n                                                                                                    |
| The workflow uses a double approval step in Gmail for quality control before publishing, which can be adjusted as needed.                                                                                                                                                                                                                                                        | Approval options in Gmail node                                                                                             |
| The Dumpling AI API is used for advanced search data scraping, including People Also Ask and related searches, which enhances content relevance.                                                                                                                                                                                                                                 | Dumpling AI official API documentation                                                                                     |
| GPT-4 is prompted to generate JSON output to streamline downstream parsing and reduce errors in content handling.                                                                                                                                                                                                                                                                  | OpenAI API best practices                                                                                                  |
| For seamless email rendering, HTML formatting is used in the Gmail node to present blog content clearly to reviewers.                                                                                                                                                                                                                                                             | HTML email templates                                                                                                        |
| This workflow is ideal for automating niche blog creation and SEO content generation but requires monitoring for AI content quality and API usage quotas.                                                                                                                                                                                                                         | SEO content automation use cases                                                                                           |

---

**Disclaimer:** The provided text is derived exclusively from an automated workflow created with n8n, an integration and automation tool. This process strictly respects current content policies and contains no illegal, offensive, or protected elements. All processed data is legal and public.