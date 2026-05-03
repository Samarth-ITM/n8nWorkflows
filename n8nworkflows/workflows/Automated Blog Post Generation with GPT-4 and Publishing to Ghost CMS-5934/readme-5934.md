Automated Blog Post Generation with GPT-4 and Publishing to Ghost CMS

https://n8nworkflows.xyz/workflows/automated-blog-post-generation-with-gpt-4-and-publishing-to-ghost-cms-5934


# Automated Blog Post Generation with GPT-4 and Publishing to Ghost CMS

### 1. Workflow Overview

This workflow automates the generation and publishing of blog posts on a Ghost CMS platform by leveraging OpenAI’s GPT-4 model. It runs on a scheduled basis every 12 hours, triggering content creation, processing the AI-generated output to extract structured metadata, and finally publishing the post via Ghost CMS’s Admin API.

The workflow is logically divided into these blocks:

- **1.1 Scheduled Trigger:** Initiates the workflow every 12 hours.
- **1.2 AI Content Generation:** Uses OpenAI’s GPT-4 to create a blog post with metadata.
- **1.3 Content Parsing & Formatting:** Processes the raw AI output to extract the title, meta description, tags, and generates HTML content.
- **1.4 Ghost CMS Post Formatting:** Converts the parsed content into Ghost’s required post format (mobiledoc).
- **1.5 Publishing:** Sends the formatted post to the Ghost CMS via authenticated API request.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger

- **Overview:**  
  This block triggers the entire workflow automatically every 12 hours, ensuring regular content generation.

- **Nodes Involved:**  
  - Schedule Trigger

- **Node Details:**

  - **Schedule Trigger**  
    - *Type & Role:* n8n’s built-in scheduler node to start workflows based on time intervals.  
    - *Configuration:* Set to trigger every 12 hours at minute 34 of the hour (e.g., 00:34, 12:34).  
    - *Inputs:* None (trigger node).  
    - *Outputs:* Triggers the OpenAI node.  
    - *Edge Cases:* If n8n instance is down or paused, trigger will be missed. Timezone handling depends on n8n server config.  
    - *Version:* 1.2

- **Sticky Note:**  
  - "🚀 This workflow runs every 12 hours and creates a blog post using OpenAI."

---

#### 1.2 AI Content Generation

- **Overview:**  
  This block uses OpenAI’s GPT-4 to generate a high-quality blog post. The prompt requests a creative or thought-provoking article, including a counterpoint paragraph, plus metadata fields.

- **Nodes Involved:**  
  - OpenAI

- **Node Details:**

  - **OpenAI**  
    - *Type & Role:* LangChain OpenAI node configured to interact with GPT-4.  
    - *Configuration:*  
      - Model: "gpt-4.1-mini-2025-04-14" (a GPT-4 variant)  
      - Prompt: Generates 2–4 paragraphs of blog content + alternative perspective + metadata (title, meta description, tags).  
      - Credentials: Uses OpenAI API key stored in “OpenAi account”.  
    - *Key Variables:* Uses a fixed prompt message in the messages parameter with detailed instructions.  
    - *Input:* Triggered by Schedule Trigger node.  
    - *Output:* Raw AI response containing combined blog content and metadata.  
    - *Edge Cases:*  
      - API key invalid or quota exceeded → authentication or rate limit errors.  
      - Timeout or connectivity issues with OpenAI API.  
      - Unexpected or malformed AI output.  
    - *Version:* 1.8

- **Sticky Note:**  
  - "✍️ The AI generates content + metadata (title, tags, meta desc)."

---

#### 1.3 Content Parsing & Formatting

- **Overview:**  
  This block extracts structured data from the raw AI text output, isolating the blog post title, meta description, tags, and body content, then wraps the body in styled HTML.

- **Nodes Involved:**  
  - Code

- **Node Details:**

  - **Code**  
    - *Type & Role:* JavaScript code node to parse and transform AI output.  
    - *Configuration:*  
      - Extracts title by regex matching "**Blog Post Title:**" line.  
      - Extracts meta description similarly.  
      - Parses tags by splitting comma-separated values.  
      - Removes metadata lines from the post body.  
      - Wraps the remaining body text in a `<div>` with centered, pre-line CSS styling.  
      - Outputs JSON with `title`, `html` body content, `status` (published), `tags`, and `meta_description`.  
    - *Inputs:* Receives AI-generated content from OpenAI node.  
    - *Outputs:* Structured JSON compatible with Ghost CMS content fields.  
    - *Edge Cases:*  
      - Missing metadata fields → defaults: title "Untitled", empty meta description, empty tags array.  
      - Malformed AI output not matching regex patterns.  
      - Unexpected line breaks affecting parsing.  
    - *Version:* 2

---

#### 1.4 Ghost CMS Post Formatting

- **Overview:**  
  Converts the parsed content into Ghost’s mobiledoc JSON format, which Ghost CMS requires for rich content posts, and arranges tags appropriately.

- **Nodes Involved:**  
  - Code1

- **Node Details:**

  - **Code1**  
    - *Type & Role:* JavaScript code node that prepares the final post payload for Ghost API.  
    - *Configuration:*  
      - Takes title, status (defaulting to “published”), and tags (converted to objects with `name` property).  
      - Uses the preformatted HTML content from previous node.  
      - Creates `mobiledoc` structure embedding the HTML as a card, per Ghost’s spec version 0.3.1.  
      - Returns JSON with a `posts` array containing the post object.  
    - *Inputs:* Receives parsed post JSON from previous Code node.  
    - *Outputs:* JSON payload formatted for Ghost Admin API.  
    - *Edge Cases:*  
      - Empty or missing tags → empty tags array sent to Ghost.  
      - Invalid or malformed HTML content could cause rendering issues.  
    - *Version:* 2

---

#### 1.5 Publishing

- **Overview:**  
  Sends the final post JSON to the Ghost CMS Admin API endpoint to create and publish the blog post.

- **Nodes Involved:**  
  - HTTP Request1

- **Node Details:**

  - **HTTP Request1**  
    - *Type & Role:* HTTP Request node used to POST the blog post JSON to Ghost CMS API.  
    - *Configuration:*  
      - URL: `https://yourcustomurl.com/ghost/api/admin/posts/` (replace with actual Ghost instance URL).  
      - Method: POST  
      - Body: JSON payload from Code1 node.  
      - Headers: Content-Type application/json.  
      - Authentication: Uses predefined credentials of type `ghostAdminApi` with OAuth2 or Admin API Key (configured as “Ghost - Transcripspiracy”).  
    - *Inputs:* Receives mobiledoc post JSON from Code1.  
    - *Outputs:* API response from Ghost CMS (success or error).  
    - *Edge Cases:*  
      - Authentication failure due to invalid credentials or expired tokens.  
      - Network errors or API downtime.  
      - API validation errors if payload is malformed.  
    - *Version:* 4.2

- **Sticky Note:**  
  - "📤 Posts are sent to your Ghost CMS via authenticated API."

---

### 3. Summary Table

| Node Name       | Node Type                         | Functional Role                         | Input Node(s)     | Output Node(s)   | Sticky Note                                               |
|-----------------|----------------------------------|---------------------------------------|-------------------|------------------|-----------------------------------------------------------|
| Schedule Trigger| n8n-nodes-base.scheduleTrigger   | Initiates the workflow every 12 hours | None              | OpenAI           | 🚀 This workflow runs every 12 hours and creates a blog post using OpenAI. |
| OpenAI          | @n8n/n8n-nodes-langchain.openAi | Generates blog post content + metadata | Schedule Trigger  | Code             | ✍️ The AI generates content + metadata (title, tags, meta desc). |
| Code            | n8n-nodes-base.code              | Parses AI output into structured JSON | OpenAI            | Code1            |                                                           |
| Code1           | n8n-nodes-base.code              | Formats post into Ghost mobiledoc JSON| Code              | HTTP Request1    |                                                           |
| HTTP Request1   | n8n-nodes-base.httpRequest       | Publishes post to Ghost CMS API       | Code1             | None             | 📤 Posts are sent to your Ghost CMS via authenticated API.|
| Sticky Note     | n8n-nodes-base.stickyNote        | Notes and comments                    | None              | None             | See respective notes above                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node:**  
   - Type: Schedule Trigger  
   - Parameters: Set interval to trigger every 12 hours at minute 34.  
   - No credentials required.

2. **Create OpenAI Node:**  
   - Type: LangChain OpenAI node (@n8n/n8n-nodes-langchain.openAi)  
   - Parameters:  
     - Model ID: `gpt-4.1-mini-2025-04-14`  
     - Messages: Single static message with prompt:  
       ```
       Write a high-quality blog post on a creative or thought-provoking topic. The tone should be engaging and immersive. Length: 2–4 paragraphs.

       Then add a brief paragraph offering an alternative perspective or logical counterpoint.

       Finally, generate:
       - Blog post title
       - Meta description
       - 5 tags
       ```  
   - Credentials: Link to OpenAI API credentials (API key).  
   - Connect Schedule Trigger’s output to OpenAI’s input.

3. **Create Code Node (Parsing AI Output):**  
   - Type: Code  
   - Parameters: Use this JavaScript code to parse AI output:
     ```javascript
     const content = $json.message.content;

     // Extract Blog Post Title
     const titleMatch = content.match(/\*\*Blog Post Title:\*\*\s*(.+)/i);
     const title = titleMatch ? titleMatch[1].trim() : 'Untitled';

     // Extract Meta Description
     const metaMatch = content.match(/\*\*Meta Description:\*\*\s*(.+)/i);
     const meta = metaMatch ? metaMatch[1].trim() : '';

     // Extract Tags
     const tagsMatch = content.match(/\*\*Tags:\*\*\s*(.+)/i);
     const rawTags = tagsMatch ? tagsMatch[1] : '';
     const tags = rawTags.split(',').map(t => t.trim()).filter(t => t);

     // Strip metadata from the body
     const body = content
       .split('\n\n')
       .filter(p => !/\*\*Blog Post Title:\*\*/i.test(p) && !/\*\*Meta Description:\*\*/i.test(p) && !/\*\*Tags:\*\*/i.test(p))
       .join('\n\n');

     // Wrap in clean HTML
     const html = `<div style="text-align:center; white-space:pre-line;">${body.trim()}</div>`;

     // Return Ghost-compatible JSON
     return [
       {
         json: {
           title,
           html,
           status: 'published',
           tags,
           meta_description: meta
         }
       }
     ];
     ```
   - Connect OpenAI node output to this node input.

4. **Create Code1 Node (Format for Ghost):**  
   - Type: Code  
   - Parameters: Use this JavaScript code to format the post for Ghost API:
     ```javascript
     const title = $json.title;
     const status = $json.status || 'published';
     const tags = ($json.tags || []).map(tag => ({ name: tag }));

     // Already wrapped with styling and line breaks — use as-is
     const htmlContent = $json.html;

     const mobiledoc = JSON.stringify({
       version: '0.3.1',
       atoms: [],
       cards: [['html', { html: htmlContent }]],
       markups: [],
       sections: [[10, 0]]
     });

     return [
       {
         posts: [
           {
             title,
             mobiledoc,
             status,
             tags
           }
         ]
       }
     ];
     ```
   - Connect Code node output to this node input.

5. **Create HTTP Request1 Node (Publish to Ghost):**  
   - Type: HTTP Request  
   - Parameters:  
     - URL: `https://yourcustomurl.com/ghost/api/admin/posts/` (replace with your Ghost admin API URL)  
     - Method: POST  
     - Authentication: Use predefined credential of type `ghostAdminApi` configured with your Ghost Admin API key or OAuth2.  
     - Headers: Content-Type = application/json  
     - Body Content: Raw JSON from previous node (`={{ JSON.stringify($json) }}`)  
     - Send body and headers enabled  
   - Connect Code1 node output to HTTP Request1 input.

6. **Credential Setup:**  
   - Configure OpenAI API credentials using your API key in n8n credentials section.  
   - Configure Ghost Admin API credentials with API key or OAuth2 token for your Ghost CMS instance.

7. **(Optional) Add Sticky Notes:**  
   - Add notes describing the purpose of workflow blocks for clarity.

---

### 5. General Notes & Resources

| Note Content                                                         | Context or Link                                      |
|----------------------------------------------------------------------|-----------------------------------------------------|
| This workflow runs every 12 hours and creates a blog post using OpenAI. | Refer to Schedule Trigger node sticky note.         |
| AI generates blog content plus metadata: title, tags, meta description. | See OpenAI node sticky note for prompt details.     |
| Posts are published to Ghost CMS via authenticated API request.       | Refer to HTTP Request node sticky note.              |
| Ghost CMS requires posts in mobiledoc JSON format; see Ghost docs for mobiledoc spec: https://ghost.org/docs/api/v5/admin/#creating-a-post | Useful for extending or troubleshooting publishing.  |
| OpenAI API usage requires valid API key and quota management.         | https://platform.openai.com/account/api-keys          |

---

**Disclaimer:**  
The provided text is exclusively derived from an automated workflow built using n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.