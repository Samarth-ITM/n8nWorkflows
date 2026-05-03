Generate SEO content and marketing copy with Claude, Serper, Sheets and SendGrid

https://n8nworkflows.xyz/workflows/generate-seo-content-and-marketing-copy-with-claude--serper--sheets-and-sendgrid-13596


# Generate SEO content and marketing copy with Claude, Serper, Sheets and SendGrid

This document provides a technical breakdown of the **Jasper AI Content Creator** workflow, designed to automate high-quality SEO and marketing content generation using Claude AI, Serper, and various delivery integrations.

### 1. Workflow Overview
The workflow functions as a headless content engine. It receives a content brief via a webhook, enriches the request with live Google Search data, generates tailored copy via Claude AI, and finally distributes the results to a tracking sheet and via email.

**Logical Blocks:**
*   **1.1 Intake & Contextualization:** Receives the payload, sets configuration defaults, and fetches real-time SEO competitor data.
*   **1.2 AI Content Generation:** Dynamically builds complex prompts for various content types (Blogs, Ads, Emails, Social), executes the AI call, and parses the structured response.
*   **1.3 Data Persistence & Notification:** Logs the execution to Google Sheets, sends a delivery notification via SendGrid, and returns a JSON summary to the requester.

---

### 2. Block-by-Block Analysis

#### 2.1 Intake & Contextualization (Stage 1)
*   **Overview:** This block captures the user's requirements and gathers external SEO data to ensure the AI has context on what is currently ranking on Google.
*   **Nodes Involved:** `Receive Brief`, `Set Config`, `Fetch SERP Data`.
*   **Node Details:**
    *   **Receive Brief (Webhook):** 
        *   *Role:* Entry point. Accepts POST requests at `/content-brief`. 
        *   *Config:* Set to "Respond to Webhook" mode to allow a custom final response.
    *   **Set Config (Set):**
        *   *Role:* Data normalization and Credential storage.
        *   *Logic:* It uses expressions to provide defaults (e.g., `wordCount` defaults to 1200 if not provided). It also stores placeholders for `anthropicKey`, `serperKey`, and `sendgridKey`.
    *   **Fetch SERP Data (HTTP Request):**
        *   *Role:* Competitor research.
        *   *Config:* Sends a POST request to `google.serper.dev/search` using the `keyword` from the config. 
        *   *Edge Cases:* "Continue on Fail" is enabled to ensure the workflow proceeds even if the SEO search fails.

#### 2.2 AI Content Generation (Stage 2)
*   **Overview:** Transforms the raw input and SEO data into a finalized piece of marketing copy using advanced prompt engineering.
*   **Nodes Involved:** `Build Claude Prompt`, `Call Claude AI`, `Parse Claude Response`.
*   **Node Details:**
    *   **Build Claude Prompt (Code):**
        *   *Role:* Prompt Factory. 
        *   *Logic:* Contains JavaScript logic to switch between four templates: `blog_post`, `ad_copy`, `email_sequence`, and `social_captions`. It formats the top 4 competitor titles into the prompt to "beat" them.
    *   **Call Claude AI (HTTP Request):**
        *   *Role:* Large Language Model execution.
        *   *Config:* Connects to Anthropic’s API using model `claude-sonnet-4-20250514`. 
        *   *Parameters:* Sets `max_tokens` to 4000 and includes a system message defining Claude as a "world-class marketing copywriter."
    *   **Parse Claude Response (Code):**
        *   *Role:* Structured Data Extraction.
        *   *Logic:* Uses Regular Expressions (Regex) to find tags like `META_TITLE:`, `META_DESC:`, and `SEO_SCORE:` within the AI's text response. It also generates a URL slug and calculates an "estimated value" for the job.

#### 2.3 Data Persistence & Notification (Stage 3)
*   **Overview:** Handles the "output" phase—storing the work, notifying the user, and closing the HTTP connection.
*   **Nodes Involved:** `Save to Google Sheets`, `Send Delivery Email`, `Send Response`.
*   **Node Details:**
    *   **Save to Google Sheets:**
        *   *Role:* Audit logging.
        *   *Config:* Appends a new row to a sheet named "Content Log" using Service Account authentication.
    *   **Send Delivery Email (HTTP Request):**
        *   *Role:* Client delivery.
        *   *Config:* POST to SendGrid API. Sends a plain text email containing a 500-character preview of the generated content and the SEO metadata.
    *   **Send Response (Respond to Webhook):**
        *   *Role:* Synchronous feedback.
        *   *Output:* Returns a JSON object with `success: true`, the `jobId`, and all calculated SEO metadata back to the initial caller.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Receive Brief | Webhook | Entry Trigger | (None) | Set Config | Stage 1 - Intake |
| Set Config | Set | Parameter Setup | Receive Brief | Fetch SERP Data | Stage 1 - Intake |
| Fetch SERP Data | HTTP Request | SEO Research | Set Config | Build Claude Prompt | Stage 1 - Intake |
| Build Claude Prompt | Code | Template Engine | Fetch SERP Data | Call Claude AI | Stage 2 - Generate |
| Call Claude AI | HTTP Request | AI Content Generation | Build Claude Prompt | Parse Claude Response | Stage 2 - Generate |
| Parse Claude Response | Code | Data Extraction | Call Claude AI | Sheets, SendGrid | Stage 2 - Generate |
| Save to Google Sheets | Google Sheets | Data Logging | Parse Claude Response | Send Response | Stage 3 - Publish |
| Send Delivery Email | HTTP Request | Email Notification | Parse Claude Response | Send Response | Stage 3 - Publish |
| Send Response | Respond to Webhook | Final API Output | Sheets, SendGrid | (End) | Stage 3 - Publish |

---

### 4. Reproducing the Workflow from Scratch

1.  **Webhook Trigger:** Create a Webhook node. Set method to `POST` and path to `content-brief`. Set "Response Mode" to `When Last Node Finishes`.
2.  **Configuration:** Add a `Set` node. Create string/number fields for: `topic`, `contentType`, `keyword`, `tone`, `audience`, `wordCount`, `brand`, and `clientEmail`. Manually add your API keys for Anthropic, Serper, and SendGrid as strings here.
3.  **SEO Enrichment:** Add an HTTP Request node. Set method to `POST`. URL: `https://google.serper.dev/search`. Add a header `X-API-KEY` linked to your config. Send a JSON body: `{"q": "{{$json.keyword}}", "num": 5}`.
4.  **Prompt Logic:** Add a Code node. Use JavaScript to define a `prompts` object. Map the incoming `contentType` to specific instructions (e.g., if "blog_post", instruct to include H1, H2, and FAQ).
5.  **AI Integration:** Add an HTTP Request node for Anthropic. URL: `https://api.anthropic.com/v1/messages`. Use headers `x-api-key` and `anthropic-version: 2023-06-01`. Body should include the model `claude-sonnet-4-20250514` and the `claudePrompt` from the previous step.
6.  **Response Cleaning:** Add another Code node. Use `.match()` regex to pull out `META_TITLE`, `META_DESC`, and `SEO_SCORE` from the text returned by Claude.
7.  **External Log:** Add a Google Sheets node. Set operation to `Append`. Connect your Google Service Account and select a spreadsheet with headers matching your parsed data (Topic, Content, SEO Score, etc.).
8.  **Email Delivery:** Add an HTTP Request node for SendGrid. URL: `https://api.sendgrid.com/v3/mail/send`. Use Bearer Token auth. Construct a JSON payload that sends the `generatedContent` to the `clientEmail`.
9.  **Completion:** Add a "Respond to Webhook" node. Select "Respond with JSON" and map all the final variables (slug, meta data, content length) to the body.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| **Setup Step 1** | Add your `ANTHROPIC_API_KEY` in the Set Config node. |
| **Setup Step 2** | Configure Serper API key for SEO data (optional but recommended). |
| **Setup Step 3** | Set your Google Sheets ID and SendGrid credentials. |
| **Content Types Supported** | `blog_post`, `ad_copy`, `email_sequence`, `social_captions`. |
| **Claude Model Used** | `claude-sonnet-4-20250514` |
| **Personalization** | For email sequences, the workflow uses `{{first_name}}` as a placeholder. |