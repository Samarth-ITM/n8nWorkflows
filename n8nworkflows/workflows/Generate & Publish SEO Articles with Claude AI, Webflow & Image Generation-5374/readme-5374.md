Generate & Publish SEO Articles with Claude AI, Webflow & Image Generation

https://n8nworkflows.xyz/workflows/generate---publish-seo-articles-with-claude-ai--webflow---image-generation-5374


# Generate & Publish SEO Articles with Claude AI, Webflow & Image Generation

### 1. Workflow Overview

This workflow automates the generation and publishing of SEO-optimized articles using AI-powered content creation, image generation, and publishing tools integrated via n8n. Its main purpose is to streamline content production for a website (productai.photo), from idea retrieval through article rewriting and thumbnail creation to publishing on Webflow and updating tracking sheets.

The workflow is logically divided into three main blocks:

- **1.1 Content Production**: Triggering the workflow periodically, fetching article ideas from Google Sheets, scraping and cleaning original content, analyzing page structure, generating SEO-optimized rewritten articles using Claude AI, and validating article quality.
  
- **1.2 Image Generation and Thumbnail Creation**: Creating descriptive prompts for AI image generation, invoking an external image generation API, retrieving the resulting image URL, and preparing it for publishing.

- **1.3 Content Publishing and Post-Processing**: Publishing the final article and thumbnail into Webflow, updating Google Sheets to mark the article as completed, and optionally notifying via Slack.

Each block consists of tightly connected nodes with clear data dependencies, ensuring a robust, end-to-end automated SEO article pipeline.

---

### 2. Block-by-Block Analysis

#### 2.1 Content Production

**Overview:**  
This block initiates the workflow, retrieves pending article ideas from Google Sheets, scrapes their content, analyzes page structure for SEO elements, rewrites the article using Claude AI, and validates content quality before proceeding.

**Nodes Involved:**  
- Schedule Trigger  
- Google Sheets  
- HTTP Request  
- Code  
- Page structure analiser  
- Content writer  
- Article written? (If node)  
- Execute Workflow  
- Is it good enough? (If node)  
- Prompt engineer  
- Auto-fixing Output Parser  
- Structured Output Parser1  

**Node Details:**

- **Schedule Trigger**  
  - Type: Trigger  
  - Role: Starts workflow every 2 hours.  
  - Config: Interval set to every 2 hours.  
  - Inputs: None  
  - Outputs: Connects to Google Sheets to fetch article ideas.  
  - Edge cases: Timing errors or trigger missed executions.

- **Google Sheets**  
  - Type: Data retrieval  
  - Role: Reads rows from the "Blogs (ideas)" sheet where 'Completed' is false.  
  - Config: OAuth2 credentials, filter on 'Completed' = false.  
  - Inputs: From Schedule Trigger  
  - Outputs: To HTTP Request (to fetch article HTML).  
  - Edge cases: API limits, auth errors, empty result sets.

- **HTTP Request**  
  - Type: HTTP client  
  - Role: Fetches the raw HTML content of article URLs from Google Sheets.  
  - Config: URL dynamically set from sheet row.  
  - Inputs: From Google Sheets  
  - Outputs: To Code node for HTML cleaning.  
  - Edge cases: HTTP errors, timeouts, invalid URLs.

- **Code**  
  - Type: JavaScript processor  
  - Role: Cleans raw HTML by trimming to `<body>`, removing `<header>`, `<script>`, and `<iframe>` tags.  
  - Config: Custom JS code for string manipulation.  
  - Inputs: From HTTP Request  
  - Outputs: To Page structure analiser  
  - Edge cases: HTML malformed, missing tags.

- **Page structure analiser**  
  - Type: LangChain agent (Claude AI)  
  - Role: Analyzes cleaned HTML to extract bullet points of page sections and SEO key points per section.  
  - Config: Custom prompt defining SEO expertise and output format.  
  - Inputs: From Code node  
  - Outputs: To Content writer  
  - Edge cases: AI model errors, network issues.

- **Content writer**  
  - Type: LangChain agent (Claude AI)  
  - Role: Rewrites the article into an SEO-optimized HTML markdown structure with a summary, following strict formatting and keyword usage rules.  
  - Config: Prompt uses primary keyword from Google Sheets, enforces JSON output, disallows commentary.  
  - Inputs: From Page structure analiser and Google Sheets  
  - Outputs: To Article written? node  
  - Edge cases: AI failure, empty output, parsing errors.

- **Article written? (If node)**  
  - Type: Conditional  
  - Role: Checks if the 'article' field in AI output is not empty to proceed.  
  - Inputs: From Content writer  
  - Outputs: Yes -> Execute Workflow, No -> (implicit failure handling)  
  - Edge cases: False negatives if output malformed.

- **Execute Workflow**  
  - Type: Sub-workflow executor  
  - Role: Runs a sub-workflow "Shape workflows — SEO Content evaluator" to assess article quality metrics.  
  - Config: Passes article and summary as inputs.  
  - Inputs: From Article written?  
  - Outputs: To Is it good enough? node  
  - Edge cases: Sub-workflow failure, input mismatches.

- **Is it good enough? (If node)**  
  - Type: Conditional  
  - Role: Evaluates quality scores (percent_ok) across multiple criteria; thresholds set at 50-60%.  
  - Inputs: From Execute Workflow  
  - Outputs: Yes -> Prompt engineer, No -> Content writer (retry)  
  - Edge cases: Missing metrics, incorrect JSON paths.

- **Prompt engineer**  
  - Type: LangChain agent (OpenAI Mini model)  
  - Role: Generates a concise, visually descriptive prompt for AI image generation based on rewritten article content.  
  - Config: Custom prompt emphasizing clarity and detail.  
  - Inputs: From Is it good enough?  
  - Outputs: To HTTP Request1 for image generation  
  - Edge cases: AI errors, output formatting issues.

- **Auto-fixing Output Parser**  
  - Type: Output parser for LangChain  
  - Role: Ensures AI outputs conform to expected JSON schema, attempts auto-correction on errors.  
  - Config: Custom instructions with error feedback loop.  
  - Inputs: From OpenAI Mini model  
  - Outputs: To Content writer (retry on error)  
  - Edge cases: Persistent parsing failures.

- **Structured Output Parser1**  
  - Type: Structured JSON parser  
  - Role: Extracts and validates structured JSON fields from AI outputs including article and summary.  
  - Config: Example JSON schema defining required fields.  
  - Inputs: From Auto-fixing Output Parser  
  - Outputs: To Auto-fixing Output Parser (loop) or Content writer  
  - Edge cases: Incorrect or incomplete JSON.

---

#### 2.2 Image Generation and Thumbnail Creation

**Overview:**  
This block generates an eye-catching thumbnail image prompt, sends it to an AI image generation API, waits for processing, retrieves the image URL, and prepares it for publication.

**Nodes Involved:**  
- Prompt engineer  
- HTTP Request1  
- Wait  
- get_image_url  

**Node Details:**

- **Prompt engineer** (also part of Content Production, described above)  
  - Generates the thumbnail prompt based on article text.  

- **HTTP Request1**  
  - Type: HTTP client  
  - Role: Sends a POST request to Replicate API to generate an image from the prompt.  
  - Config: JSON body contains prompt and generation parameters (aspect ratio 16:9, num_inference_steps=50, guidance_scale=3.5).  
  - Headers include Bearer token for authentication and 'Prefer: wait' to wait synchronously.  
  - Inputs: From Prompt engineer  
  - Outputs: To Wait node  
  - Edge cases: API rate limits, token expiry, network errors.

- **Wait**  
  - Type: Delay  
  - Role: Waits 30 seconds to ensure the image generation process completes.  
  - Inputs: From HTTP Request1  
  - Outputs: To get_image_url  
  - Edge cases: Too short/long delay causing timing issues.

- **get_image_url**  
  - Type: HTTP client  
  - Role: Retrieves the generated image URL from the API response.  
  - Config: GET request with Authorization header using Bearer token.  
  - Inputs: From Wait  
  - Outputs: To Slack and Webflow publishing nodes  
  - Edge cases: Authorization failures, missing URL field.

---

#### 2.3 Content Publishing and Post-Processing

**Overview:**  
This block publishes the article and image to Webflow, updates Google Sheets to mark the article as completed, and optionally sends a Slack notification.

**Nodes Involved:**  
- Slack (disabled)  
- Google Sheets1  
- Webflow  
- No Operation, do nothing  

**Node Details:**

- **Slack**  
  - Type: Slack message sender  
  - Role: Sends a formatted Slack message announcing the new article with thumbnail image.  
  - Config: OAuth2 authentication, channel set, message uses blocks JSON with article title and image URL.  
  - Inputs: From get_image_url  
  - Outputs: To Google Sheets1 and Webflow  
  - Disabled: True (not active)  
  - Edge cases: Slack API permissions, message formatting errors.

- **Google Sheets1**  
  - Type: Data update  
  - Role: Marks the article row as completed in Google Sheets by updating 'Completed' to true using row_number as key.  
  - Config: OAuth2 credentials, update mapping for 'Completed' and 'row_number'.  
  - Inputs: From Slack  
  - Outputs: To No Operation node (end chain)  
  - Edge cases: API update failures, concurrency issues.

- **Webflow**  
  - Type: Webflow API node  
  - Role: Creates a new article entry in the Webflow collection with the generated content and image.  
  - Config: OAuth2 credentials, live publish enabled, fields populated from AI outputs and image URL, including article body, summary, and thumbnail.  
  - Inputs: From Slack  
  - Outputs: To No Operation node  
  - Edge cases: Webflow API limits, field validation errors.

- **No Operation, do nothing**  
  - Type: Placeholder/end node  
  - Role: Terminates the workflow branch gracefully after publishing and update steps.  
  - Inputs: From Google Sheets1 and Webflow  
  - Outputs: None  

---

### 3. Summary Table

| Node Name             | Node Type                          | Functional Role                            | Input Node(s)                | Output Node(s)                         | Sticky Note                                |
|-----------------------|----------------------------------|--------------------------------------------|-----------------------------|---------------------------------------|--------------------------------------------|
| Schedule Trigger      | Schedule Trigger                 | Initiates workflow every 2 hours           | None                        | Google Sheets                         |                                            |
| Google Sheets         | Google Sheets                   | Fetches article ideas not yet completed    | Schedule Trigger            | HTTP Request                         |                                            |
| HTTP Request          | HTTP Request                   | Fetches raw HTML of article URLs            | Google Sheets               | Code                                |                                            |
| Code                  | Code                           | Cleans HTML, removing header/scripts/etc   | HTTP Request                | Page structure analiser              |                                            |
| Page structure analiser| LangChain Agent (Claude AI)    | Analyzes page sections and SEO points       | Code                       | Content writer                      |                                            |
| Content writer        | LangChain Agent (Claude AI)    | Rewrites article SEO-optimized in JSON      | Page structure analiser, Google Sheets | Article written?                   | Sticky Note: Content writer block          |
| Article written?       | If                            | Checks if article output is valid            | Content writer              | Execute Workflow (Yes), none (No)  |                                            |
| Execute Workflow      | Execute Workflow               | Runs SEO content evaluation sub-workflow    | Article written?            | Is it good enough?                  |                                            |
| Is it good enough?     | If                            | Validates SEO quality scores                 | Execute Workflow            | Prompt engineer (Yes), Content writer (No) | Sticky Note: SEO quality validation       |
| Prompt engineer       | LangChain Agent (OpenAI Mini) | Creates AI prompt for image generation       | Is it good enough?          | HTTP Request1                      |                                            |
| HTTP Request1         | HTTP Request                   | Sends prompt to image generation API         | Prompt engineer             | Wait                              |                                            |
| Wait                  | Wait                          | Waits for image generation completion        | HTTP Request1               | get_image_url                     |                                            |
| get_image_url         | HTTP Request                   | Retrieves generated image URL                 | Wait                       | Slack                             | Sticky Note: Content publishing            |
| Slack                 | Slack                         | (Disabled) Sends Slack notification          | get_image_url               | Google Sheets1, Webflow            |                                            |
| Google Sheets1        | Google Sheets                 | Updates article row to completed              | Slack                      | No Operation                     |                                            |
| Webflow               | Webflow                       | Publishes article and image content           | Slack                      | No Operation                     |                                            |
| No Operation, do nothing| NoOp                         | Ends the publishing branch                     | Google Sheets1, Webflow     | None                             |                                            |
| Auto-fixing Output Parser| LangChain Output Parser      | Ensures AI output JSON is valid               | OpenAI Mini model           | Content writer                   |                                            |
| Structured Output Parser1| LangChain Output Parser     | Parses AI JSON output structure               | Auto-fixing Output Parser   | Auto-fixing Output Parser        |                                            |
| Haiku                 | LangChain Agent (Anthropic)   | (Unused) Possibly for experimental use       |                             | Page structure analiser           |                                            |
| Thinking Claude       | LangChain Agent (Anthropic)   | (Unused) Possibly alternative content writer |                             | Content writer                   |                                            |
| Get Articles          | Webflow Tool                  | Retrieves existing Webflow articles for internal linking |                             | Content writer                   |                                            |
| Send error            | Slack                         | Sends error message if content writer produces nothing |                             | None                             |                                            |
| Sticky Notes (various)| Sticky Note                   | Visual comments on workflow parts             |                             |                                  | Content production, SEO validation, publishing blocks |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node**  
   - Type: Schedule Trigger  
   - Set to run every 2 hours.  
   - Position: Start of workflow.

2. **Create Google Sheets Node**  
   - Type: Google Sheets  
   - Connect input from Schedule Trigger.  
   - Use OAuth2 credentials for Google Sheets.  
   - Configure to read from document ID `16rMg45cnPeJMoaCQmmhF5dvsbpkwtINUTx57WEkcDDM`, sheet "Blogs (ideas)" (gid 268886635).  
   - Filter rows where 'Completed' column is false.

3. **Create HTTP Request Node**  
   - Connect from Google Sheets.  
   - Set URL dynamically from Google Sheets row field `URL`.  
   - Use GET method, default options.

4. **Create Code Node**  
   - Connect from HTTP Request.  
   - Add JS code to trim HTML to `<body>`, remove `<header>`, `<script>`, `<iframe>` tags.  
   - Return modified HTML in `item.json.data`.

5. **Create Page structure analiser Node**  
   - LangChain Agent node using Claude AI credentials.  
   - Connect from Code node.  
   - Prompt instructs to extract page sections and SEO key points as bullet points from cleaned HTML (`$json.data`).  
   - Configure to parse AI output.

6. **Create Content writer Node**  
   - LangChain Agent node using Claude AI credentials.  
   - Connect from Page structure analiser and Google Sheets (for keyword input).  
   - Prompt instructs to rewrite article in SEO-optimized JSON with "article" and "summary" fields, using primary keyword from Google Sheets.  
   - Output expected as structured JSON.

7. **Create Article written? If Node**  
   - Connect from Content writer.  
   - Condition: Check if `output.article` field is not empty.  
   - Yes path to Execute Workflow node, No path to error handling or retry.

8. **Create Execute Workflow Node**  
   - Connect from Article written? (Yes).  
   - Configure to run sub-workflow "Shape workflows — SEO Content evaluator" with inputs: article and summary from Content writer output.

9. **Create Is it good enough? If Node**  
   - Connect from Execute Workflow.  
   - Condition: Check SEO quality metrics (percent_ok fields) meet thresholds (e.g. >60% for main checks, >50% for others).  
   - Yes path to Prompt engineer node, No path to Content writer for retry.

10. **Create Prompt engineer Node**  
    - LangChain Agent node using OpenAI Mini model credentials.  
    - Connect from Is it good enough? (Yes).  
    - Prompt to generate detailed image generation prompt based on article content.

11. **Create HTTP Request1 Node**  
    - Connect from Prompt engineer.  
    - POST request to Replicate API with JSON body containing prompt and parameters for image generation (aspect ratio 16:9, steps 50, guidance scale 3.5).  
    - Headers: Authorization Bearer token, Prefer: wait.

12. **Create Wait Node**  
    - Connect from HTTP Request1.  
    - Wait for 30 seconds.

13. **Create get_image_url Node**  
    - Connect from Wait.  
    - HTTP GET request to retrieve generated image URL.  
    - Authorization header with Bearer token from environment or credentials.

14. **Create Slack Node (optional, currently disabled)**  
    - Connect from get_image_url.  
    - OAuth2 Slack credentials.  
    - Send formatted message with article title and thumbnail image to specified Slack channel.

15. **Create Google Sheets1 Node**  
    - Connect from Slack node.  
    - Update original Google Sheets row's 'Completed' field to true using row_number key.

16. **Create Webflow Node**  
    - Connect from Slack node.  
    - OAuth2 credentials for Webflow.  
    - Create new item in Webflow collection "64b1bae9c2d06f1241365376" with fields: cover-image, name, article-body-text, read-time, short-paragraph, first-post-image populated from AI outputs and image URL.

17. **Create No Operation Node**  
    - Connect from Google Sheets1 and Webflow nodes to gracefully end workflow.

18. **Create Auto-fixing Output Parser Node**  
    - Connect output from OpenAI Mini model (Prompt engineer) to this node.  
    - Configure with instructions to ensure JSON output matches schema, auto-correcting errors.

19. **Create Structured Output Parser1 Node**  
    - Connect output from Auto-fixing Output Parser.  
    - Define JSON schema for article and summary fields.

20. **Configure Credentials**  
    - Google Sheets OAuth2 with appropriate scopes.  
    - Slack OAuth2 with messaging permissions.  
    - Webflow OAuth2 with collection write permissions.  
    - OpenAI API key for Mini model.  
    - Anthropic API key for Claude models (if used).  
    - Replicate API Bearer token for image generation.

21. **Sub-workflow Setup**  
    - Ensure the sub-workflow "Shape workflows — SEO Content evaluator" accepts two inputs: article (string) and summary (string).  
    - Outputs a JSON with percent_ok metrics used for quality validation.

---

### 5. General Notes & Resources

| Note Content                                                                                                     | Context or Link                                             |
|-----------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------|
| The workflow uses Claude AI (Anthropic) models for SEO rewriting, requiring Anthropic API credentials.          | https://www.anthropic.com/                                  |
| Replicate API is used for AI image generation; ensure API token is valid and has required scopes.              | https://replicate.com/                                       |
| Webflow API integration requires OAuth2 token with CMS write permissions.                                       | https://developers.webflow.com/                             |
| Slack node is currently disabled but can be enabled to notify team of published articles with thumbnails.      | Slack API documentation: https://api.slack.com/            |
| The AI prompts enforce strict JSON output to facilitate parsing and reduce errors in downstream nodes.        | Prompt engineering best practices                            |
| Sub-workflow "Shape workflows — SEO Content evaluator" is critical for quality gating; must be implemented separately. | Internal project workflow                                    |
| Sticky notes in the original workflow visually group nodes into blocks: "Content production", "SEO quality validation", and "Content publishing". | Visual workflow organization                                 |

---

**Disclaimer:** The provided reference is based exclusively on an automated n8n workflow. It complies with content policies and contains no illegal or protected content. All data handled is lawful and publicly accessible.