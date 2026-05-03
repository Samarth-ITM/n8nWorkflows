Automated Social Media Content Creation with OpenAI, LinkedIn & Twitter Approval

https://n8nworkflows.xyz/workflows/automated-social-media-content-creation-with-openai--linkedin---twitter-approval-6486


# Automated Social Media Content Creation with OpenAI, LinkedIn & Twitter Approval

### 1. Workflow Overview

This workflow, titled **Automated Social Media Content Creation with OpenAI, LinkedIn & Twitter Approval**, is designed to automate the creation, approval, and publishing of social media posts tailored for LinkedIn and Twitter (X). It targets content creators, marketing teams, or founders who want to streamline social media management by generating platform-optimized posts using AI, obtaining approval via email, and then publishing approved content automatically.

The workflow is logically structured into five main blocks:

- **1.1 Input Reception:** Collect user input via a web form specifying the topic and optional image preferences.
- **1.2 AI-Driven Content Generation:** Use OpenAI language models to generate platform-specific posts and structure the output according to a defined JSON schema.
- **1.3 Email-Based Approval Process:** Format the generated content into an HTML email, send it to an approver, and wait for approval.
- **1.4 Visual Content Generation & Post-Publishing:** Based on image preferences, generate or fetch images, then post approved content to LinkedIn and Twitter.
- **1.5 Status Updates & Logging:** Notify via Slack about live posts and log published posts to Google Sheets.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:** Captures user input regarding the social media post topic, optional keywords, hashtags, link, and image preferences through a web form.
- **Nodes Involved:**  
  - Submit Social Post Details

- **Node Details:**

  - **Submit Social Post Details**  
    - Type: Form Trigger  
    - Role: Entry point for user input; triggers workflow upon form submission.  
    - Configuration:  
      - Form fields: Topic (required), Keywords/Hashtags (optional), Link (optional), Image option (dropdown: AI generated or Image Link).  
      - Button label: "Generate Social Media Content"  
      - Form description guides the user on input expectations.  
    - Input: User-submitted form data  
    - Output: JSON with user inputs for downstream processing  
    - Edge cases: Missing required Topic field will prevent submission; optional fields may be empty.  
    - Version-specific: Uses formTrigger node v2.2.

#### 2.2 AI-Driven Content Generation

- **Overview:** Generates tailored social media post text for LinkedIn and Twitter using OpenAI models; parses and structures output as per a JSON schema.
- **Nodes Involved:**  
  - gpt-4o LLM  
  - Social Media Content Factory  
  - gpt-4o-mini  
  - Social Media Content  
  - Prepare Content Review Email

- **Node Details:**

  - **gpt-4o LLM**  
    - Type: LangChain OpenAI Chat LLM  
    - Role: Primary AI model for generating raw social media content text based on the user input topic.  
    - Configuration: Model set to "gpt-4o", response format "text".  
    - Credentials: OpenAI API key required.  
    - Input: User topic and parameters fed as prompt.  
    - Output: Raw AI-generated text for content factory.  
    - Edge cases: API rate limits, timeout, or invalid prompt errors.

  - **Social Media Content Factory**  
    - Type: LangChain Agent  
    - Role: Processes raw AI output into platform-optimized posts (LinkedIn and Twitter) with detailed instructions on tone, style, hashtags, and CTAs.  
    - Configuration: Contains detailed system instructions specifying style, tone, hashtags, CTAs, and output JSON schema.  
    - Input: Text from gpt-4o LLM plus user input fields.  
    - Output: Structured JSON with platform posts and metadata.  
    - Edge cases: Parsing failures, model response inconsistencies.

  - **gpt-4o-mini**  
    - Type: LangChain OpenAI Chat LLM  
    - Role: Generates a clean, modern HTML email content for content review based on the structured JSON output from Social Media Content Factory.  
    - Configuration: Model set to "gpt-4o-mini", response format "text".  
    - Input: Structured JSON output from Social Media Content Factory.  
    - Output: HTML email body.  
    - Edge cases: API errors, malformed HTML output.

  - **Social Media Content**  
    - Type: LangChain Structured Output Parser  
    - Role: Parses the AI-generated content strictly following the defined JSON schema for social media posts.  
    - Configuration: Manual schema defining expected JSON structure of name, description, platform posts, hashtags, CTAs, and notes.  
    - Input: AI model raw output text.  
    - Output: Parsed JSON data for further processing.  
    - Edge cases: Schema mismatch, parsing errors.

  - **Prepare Content Review Email**  
    - Type: LangChain Agent  
    - Role: Converts structured post data into a visually formatted HTML email for approval.  
    - Configuration: Prompt details specify email layout with tables, inline CSS, font styles, and platform-specific formatting.  
    - Input: Parsed JSON from Social Media Content Factory.  
    - Output: HTML string for email body.  
    - Edge cases: Formatting inconsistencies, missing data fields.

#### 2.3 Email-Based Approval Process

- **Overview:** Sends the formatted content in an email to approvers, waits for approval response, and branches workflow accordingly.
- **Nodes Involved:**  
  - Gmail User for Approval  
  - Is Content Approved?

- **Node Details:**

  - **Gmail User for Approval**  
    - Type: Gmail node (sendAndWait)  
    - Role: Sends the content review email to specified email addresses and waits for a double approval response.  
    - Configuration:  
      - SendTo: dynamic email address from input data  
      - Subject: Includes post name and description dynamically.  
      - Message: HTML email content from Prepare Content Review Email.  
      - Approval options: Double approval type with 45 minutes wait time limit.  
    - Credentials: Gmail OAuth2 required.  
    - Input: HTML email content and recipient email.  
    - Output: Approval status JSON including boolean approved flag.  
    - Edge cases: Email delivery failures, timeout waiting for approval, user declines.

  - **Is Content Approved?**  
    - Type: If node  
    - Role: Branches workflow based on approval boolean flag.  
    - Configuration: Checks if `$json.data.approved` is true.  
    - Input: Approval response from Gmail User for Approval.  
    - Output: If true, proceeds; else stops or alternative path (not defined).  
    - Edge cases: Missing approval flag, malformed input.

#### 2.4 Visual Content Generation & Post-Publishing

- **Overview:** Depending on user choice, generates AI images or fetches existing images, then publishes approved posts to LinkedIn and Twitter.
- **Nodes Involved:**  
  - If (image choice)  
  - Generate an image  
  - HTTP Request  
  - LinkedIn Post  
  - Create Tweet  
  - Create a post  
  - LinkedIn Post (duplicate node name)  
  - Send a message

- **Node Details:**

  - **If (image choice)**  
    - Type: If node  
    - Role: Routes based on whether the user selected "AI generated" image or "Image Link".  
    - Configuration: Compares `Submit Social Post Details` node’s Image field to "AI generated".  
    - Input: User input from form.  
    - Output: True branch to generate image, false branch to fetch image via HTTP.  
    - Edge cases: Unexpected image choice values.

  - **Generate an image**  
    - Type: OpenAI Image generation  
    - Role: Creates an AI generated image based on topic and description.  
    - Configuration: Prompt dynamically constructed from user topic and generated post description.  
    - Credentials: OpenAI API key required.  
    - Input: Topic and description strings.  
    - Output: Image binary data or URL.  
    - Edge cases: API limits, malformed prompt.

  - **HTTP Request**  
    - Type: HTTP Request node  
    - Role: Fetches image from user-provided online link (Image Link option).  
    - Configuration: URL parameter set dynamically (likely from user input).  
    - Input: URL string.  
    - Output: Fetched image data.  
    - Edge cases: Invalid URL, network timeout.

  - **LinkedIn Post** (two nodes with same name but different positions)  
    - Type: LinkedIn node  
    - Role: Posts LinkedIn content with attached images.  
    - Configuration: Uses dynamic text from Social Media Content Factory output, includes call to action and hashtags.  
    - Credentials: LinkedIn OAuth2.  
    - Input: Post text and image binary data.  
    - Output: LinkedIn post response.  
    - Edge cases: API errors, media upload failures.

  - **Create Tweet**  
    - Type: Twitter node  
    - Role: Posts tweet content with hashtags.  
    - Configuration: Text composed from Social Media Content Factory output.  
    - Credentials: Twitter OAuth2.  
    - Input: Tweet text and hashtags.  
    - Output: Tweet response.  
    - Edge cases: Character limit exceeded, API rate limits.

  - **Create a post**  
    - Type: LinkedIn node  
    - Role: Posts default or fallback LinkedIn post content.  
    - Configuration: Similar to LinkedIn Post node.  
    - Credentials: LinkedIn OAuth2.  
    - Input: Post text.  
    - Output: Post response.  
    - Edge cases: API errors.

  - **Send a message**  
    - Type: Slack node  
    - Role: Notifies Slack #social channel that LinkedIn post is live.  
    - Configuration: Text dynamically includes LinkedIn post content.  
    - Credentials: Slack API token.  
    - Input: Post text.  
    - Output: Slack message response.  
    - Edge cases: Slack API failures.

#### 2.5 Status Updates & Logging

- **Overview:** Logs published posts to Google Sheets and updates status via messaging.
- **Nodes Involved:**  
  - Append or update row in sheet  
  - Send a message (Slack) [already described]

- **Node Details:**

  - **Append or update row in sheet**  
    - Type: Google Sheets node  
    - Role: Logs the tweet text into a specific Google Sheet for record-keeping.  
    - Configuration:  
      - Operation: appendOrUpdate  
      - Sheet and document specified by ID and sheet name.  
      - Maps "Post" column to tweet text.  
    - Credentials: Google Sheets OAuth2 required.  
    - Input: Tweet text from Create Tweet node.  
    - Output: Sheet update response.  
    - Edge cases: Sheet access errors, rate limits.

---

### 3. Summary Table

| Node Name                 | Node Type                                     | Functional Role                                  | Input Node(s)                     | Output Node(s)                                | Sticky Note                                                                                          |
|---------------------------|-----------------------------------------------|-------------------------------------------------|----------------------------------|-----------------------------------------------|----------------------------------------------------------------------------------------------------|
| Submit Social Post Details | Form Trigger                                  | User input collection for post topic & options | None                             | Social Media Content Factory                   |                                                                                                    |
| gpt-4o LLM                | LangChain OpenAI Chat LLM                     | Generate raw social media content                | Submit Social Post Details        | Social Media Content Factory                   |                                                                                                    |
| Social Media Content Factory| LangChain Agent                              | Refines and structures AI content per platform  | gpt-4o LLM                      | Prepare Content Review Email                    | Detailed prompt defining style, tone, SEO, hashtags, platform-specific CTAs                         |
| Social Media Content       | LangChain Output Parser Structured            | Parses AI output to structured JSON              | Social Media Content Factory     | Prepare Content Review Email                    |                                                                                                    |
| gpt-4o-mini               | LangChain OpenAI Chat LLM                     | Generate HTML email content for approval         | Social Media Content Factory     | Gmail User for Approval                         |                                                                                                    |
| Prepare Content Review Email| LangChain Agent                              | Format content as HTML email for approval        | Social Media Content Factory     | Gmail User for Approval                         |                                                                                                    |
| Gmail User for Approval    | Gmail (sendAndWait)                           | Send approval email and wait for response        | Prepare Content Review Email     | Is Content Approved?                           |                                                                                                    |
| Is Content Approved?       | If Node                                      | Branch based on approval boolean                  | Gmail User for Approval          | If (image choice)                              |                                                                                                    |
| If (image choice)          | If Node                                      | Route to image generation or image fetch          | Is Content Approved?             | Generate an image / HTTP Request                |                                                                                                    |
| Generate an image          | OpenAI Image Generation                       | Generate AI image for post                         | If (image choice) True branch    | LinkedIn Post, Create Tweet                    |                                                                                                    |
| HTTP Request              | HTTP Request                                  | Fetch image from existing URL                      | If (image choice) False branch   | Create a post                                  |                                                                                                    |
| LinkedIn Post             | LinkedIn node                                 | Publish LinkedIn post with image                   | Generate an image                | Send a message                                 |                                                                                                    |
| Create Tweet              | Twitter node                                  | Publish Tweet                                     | Generate an image                | Append or update row in sheet                   |                                                                                                    |
| Create a post             | LinkedIn node                                 | Publish LinkedIn post from HTTP fetched image     | HTTP Request                   | Send a message                                 |                                                                                                    |
| Send a message            | Slack node                                    | Notify Slack channel of live LinkedIn post        | LinkedIn Post, Create a post     | None                                          |                                                                                                    |
| Append or update row in sheet| Google Sheets node                         | Log published Tweet text                           | Create Tweet                    | None                                          |                                                                                                    |
| Sticky Note2              | Sticky Note                                   | Step 2: Send approval Email                        | None                           | None                                          |                                                                                                    |
| Sticky Note6              | Sticky Note                                   | From Idea to Post: Social Media Automation overview| None                           | None                                          | Perfect for founders or marketers to save time & ensure consistent branding                        |
| Sticky Note10             | Sticky Note                                   | Step 1 : Generate social media post - text        | None                           | None                                          |                                                                                                    |
| Sticky Note11             | Sticky Note                                   | Step 3: Generate social media post - image or choose existing | None                  | None                                          |                                                                                                    |
| Sticky Note12             | Sticky Note                                   | Step 4: Publish posts                              | None                           | None                                          |                                                                                                    |
| Sticky Note13             | Sticky Note                                   | Step 5: Update status via messaging apps/sheets/docs| None                         | None                                          |                                                                                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Form Trigger node:**  
   - Name: `Submit Social Post Details`  
   - Configure form fields: Topic (required), Keywords/Hashtags (optional), Link (optional), Image choice (dropdown: "AI generated", "Image Link").  
   - Set button label: "Generate Social Media Content"  
   - Description: Brief instructions on input expectations.

2. **Add an OpenAI Chat LLM node:**  
   - Name: `gpt-4o LLM`  
   - Model: `gpt-4o`  
   - Response format: `text`  
   - Connect input from `Submit Social Post Details`.  
   - Set OpenAI credentials.

3. **Add a LangChain Agent node:**  
   - Name: `Social Media Content Factory`  
   - Paste the detailed system prompt defining content style, tone, hashtags, platform-specific requirements, and output JSON schema.  
   - Input: Output from `gpt-4o LLM`.  
   - Enable output parser.

4. **Add LangChain Output Parser Structured node:**  
   - Name: `Social Media Content`  
   - Configure manual JSON schema matching expected AI output (including LinkedIn and Twitter posts, hashtags, CTAs).  
   - Input: Output from `Social Media Content Factory`.

5. **Add another OpenAI Chat LLM node:**  
   - Name: `gpt-4o-mini`  
   - Model: `gpt-4o-mini`  
   - Response format: `text`  
   - Input: Output from `Social Media Content Factory`.

6. **Add LangChain Agent node for email formatting:**  
   - Name: `Prepare Content Review Email`  
   - Paste prompt to generate HTML email with table layout, inline CSS, styling, and platform cards as per instructions.  
   - Input: Output from `Social Media Content Factory`.

7. **Create Gmail node:**  
   - Name: `Gmail User for Approval`  
   - Operation: `sendAndWait`  
   - Send to: Dynamic email address (from input or predefined).  
   - Subject: Include post name and description dynamically.  
   - Message body: HTML from `Prepare Content Review Email`.  
   - Approval options: Double approval, 45 minutes wait.  
   - Connect input from `Prepare Content Review Email`.  
   - Set Gmail OAuth2 credentials.

8. **Add If node:**  
   - Name: `Is Content Approved?`  
   - Condition: Check if approval response `$json.data.approved` is true.  
   - Connect input from `Gmail User for Approval`.

9. **Add If node:**  
   - Name: `If (image choice)`  
   - Condition: Compare `Submit Social Post Details` Image field to "AI generated".  
   - Connect input from `Is Content Approved?` (true branch).

10. **Add OpenAI Image generation node:**  
    - Name: `Generate an image`  
    - Prompt: Compose prompt using user topic and content description.  
    - Connect input from `If (image choice)` true branch.  
    - Set OpenAI credentials.

11. **Add HTTP Request node:**  
    - Name: `HTTP Request`  
    - Purpose: Fetch image from provided URL (for "Image Link" choice).  
    - URL: Dynamic from user input.  
    - Connect input from `If (image choice)` false branch.

12. **Add LinkedIn Post node:**  
    - Name: `LinkedIn Post`  
    - Text: Use LinkedIn post from content factory output including CTAs and hashtags.  
    - Media: Attach image from either Generate an image or HTTP Request node.  
    - Credentials: LinkedIn OAuth2.  
    - Connect input from `Generate an image` and `HTTP Request`.

13. **Add Twitter Post node:**  
    - Name: `Create Tweet`  
    - Text: Use Twitter post and hashtags from content factory output.  
    - Credentials: Twitter OAuth2.  
    - Connect input from `Generate an image`.

14. **Add another LinkedIn Post node:**  
    - Name: `Create a post` (fallback post)  
    - Similar config to LinkedIn Post, connected from `HTTP Request`.

15. **Add Slack node:**  
    - Name: `Send a message`  
    - Message: Notify channel with live post details.  
    - Channel: `#social` or defined Slack channel.  
    - Credentials: Slack API token.  
    - Connect input from LinkedIn Post nodes.

16. **Add Google Sheets node:**  
    - Name: `Append or update row in sheet`  
    - Operation: Append or update tweets in a sheet.  
    - Map tweet text to "Post" column.  
    - Credentials: Google Sheets OAuth2.  
    - Connect input from `Create Tweet`.

17. **Connect nodes logically:**  
    - Submit Social Post Details → gpt-4o LLM → Social Media Content Factory → Social Media Content → Prepare Content Review Email → Gmail User for Approval → Is Content Approved? → If (image choice) → [Generate an image OR HTTP Request] → LinkedIn Post/Create a post & Create Tweet → Send a message → Append or update row in sheet.

---

### 5. General Notes & Resources

| Note Content                                                                                                                         | Context or Link                                                                                          |
|--------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------|
| The workflow automates social media content creation, approval, and publishing for LinkedIn and Twitter using OpenAI and OAuth2 APIs.| Workflow purpose and integration overview.                                                             |
| Email approval uses Gmail node’s sendAndWait with double approval to ensure content quality control.                                  | Gmail node documentation: https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.gmail/   |
| LinkedIn and Twitter OAuth2 credentials must be preconfigured with appropriate API permissions for posting.                         | LinkedIn API: https://docs.microsoft.com/en-us/linkedin/shared/integrations/people/profile-api          |
| OpenAI API used for both text generation and image creation; ensure quota availability and error handling for rate limits.           | OpenAI API docs: https://platform.openai.com/docs/api-reference                                          |
| Slack notifications provide real-time updates of published posts, facilitating team communication.                                   | Slack API docs: https://api.slack.com/                                                                  |
| Google Sheets node appends published tweets for archival and analytics.                                                               | Google Sheets API: https://developers.google.com/sheets/api                                              |
| Sticky notes provide stepwise guidance inside the workflow UI for clarity on process stages.                                           | Useful for onboarding and maintenance.                                                                  |

---

**Disclaimer:** This document is generated from an n8n workflow JSON export and reflects the design and logic of the automated social media content creation process. All data and processes comply with content policies and legal standards.