Generate research-backed infographics with GPT-4, kie.ai, and Gmail

https://n8nworkflows.xyz/workflows/generate-research-backed-infographics-with-gpt-4--kie-ai--and-gmail-14000


# Generate research-backed infographics with GPT-4, kie.ai, and Gmail

# 1. Workflow Overview

This workflow collects infographic requirements from a web form, uses an AI agent backed by OpenAI web search to research the topic and write an image-generation prompt, sends that prompt to kie.ai to generate the infographic, polls the job until completion, and then emails either the finished image or an error notification.

It is designed for users who want research-backed infographics without manual design work. Typical use cases include marketing visuals, educational assets, consultant deliverables, and social-media-ready data summaries.

## 1.1 Input Reception

The workflow starts with an n8n Form Trigger that captures all user-defined infographic requirements: headline, topic/data points, style, layout, aspect ratio, resolution, and output format.

## 1.2 AI Prompt Engineering

The submitted form data is passed to an AI Agent node. That agent uses an OpenAI chat model with built-in web search to research the topic and transform the request into a production-ready image prompt optimized for infographic generation.

## 1.3 Image Job Submission and Polling

The generated prompt is submitted to the kie.ai API as an asynchronous image-generation task. The workflow then waits 15 seconds and repeatedly checks job status until the job succeeds, fails, or exceeds the retry limit.

## 1.4 Success Delivery

If the job succeeds, the generated image URL is extracted, the image is downloaded, and a Gmail node sends it as an email attachment.

## 1.5 Error Handling and Timeout Management

If job creation fails, status polling returns an API problem, the generation job explicitly fails, or polling exceeds 20 retries, the workflow builds a contextual error email and sends it via Gmail.

---

# 2. Block-by-Block Analysis

## 2.1 Block: User Input

### Overview
This block exposes a public n8n form where the user defines the infographic request. It serves as the sole entry point for the main workflow and provides all downstream variables.

### Nodes Involved
- On form submission

### Node Details

#### On form submission
- **Type and technical role:** `n8n-nodes-base.formTrigger`; workflow entry trigger receiving form submissions over an n8n-hosted form endpoint.
- **Configuration choices:**
  - Form title: **Infographics Maker**
  - Description: **Use AI to Generate Well-Researched, High-Quality Infographics**
  - Attribution disabled
  - Required fields include:
    - Main Headline
    - Key Data Points
    - Target Audience
    - Tone of Voice
    - Visualization Options
    - Art Style
    - Color Palette
    - Layout Structure
    - Font Family
    - Information Density
    - Aspect Ratio
    - Resolution
    - Output Format
- **Key expressions or variables used:** None internally, but it emits all form fields as JSON properties for downstream nodes.
- **Input and output connections:**
  - Input: none, trigger node
  - Output: `Build Image Prompt`
- **Version-specific requirements:** Type version `2.3`; behavior depends on the Form Trigger features available in the installed n8n release.
- **Edge cases or potential failure types:**
  - Form not reachable if workflow is inactive
  - Browser/client submission issues
  - Changes to field labels will break downstream expressions referencing exact labels such as `Main Headline` or `Aspect Ratio`
- **Sub-workflow reference:** None

---

## 2.2 Block: AI Prompt Engineering

### Overview
This block converts the form submission into a refined prompt for image generation. It combines a LangChain AI Agent with an OpenAI model configured for web search, enabling the prompt to be informed by online research.

### Nodes Involved
- Build Image Prompt
- Researcher

### Node Details

#### Build Image Prompt
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent`; agent node orchestrating LLM-based prompt construction.
- **Configuration choices:**
  - Prompt type: defined directly in the node
  - Strong role framing as an art director and prompt engineer
  - Uses form fields such as headline, data points, layout, visualization style, tone, style, density, font, and colors
  - Explicitly instructs the agent to use the `Researcher` model with internet access
  - Enforces output as raw prompt string only
  - Focuses on clean composition, readability, and infographic-friendly generation
- **Key expressions or variables used:**
  - `{{ $json['Main Headline'] }}`
  - `{{ $json['Key Data Points'] }}`
  - `{{ $json['Layout Structure'] }}`
  - `{{ $json['Visualization Options'] }}`
  - `{{ $json['Tone of Voice'] }}`
  - `{{ $json['Art Style'] }}`
  - `{{ $json['Information Density'] }}`
  - `{{ $json['Font Family'] }}`
  - `{{ $json['Color Palette'] }}`
- **Input and output connections:**
  - Main input: `On form submission`
  - AI language model input: `Researcher`
  - Main output: `Generate Infographic`
- **Version-specific requirements:** Type version `3`; requires a compatible n8n LangChain/AI node set.
- **Edge cases or potential failure types:**
  - LLM rate limits or authentication failures
  - Prompt output may not match expectations if field labels are changed
  - Web search tool availability may vary by OpenAI model/capability
  - Agent may produce overly long or suboptimal prompts if inputs are vague
- **Sub-workflow reference:** None

#### Researcher
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`; OpenAI chat model supplying the language model backend to the AI Agent.
- **Configuration choices:**
  - Model: `gpt-4.1-mini`
  - Built-in web search enabled
  - Search context size: `medium`
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Output via AI language model connection to `Build Image Prompt`
- **Version-specific requirements:** Type version `1.3`; requires OpenAI credentials and an n8n version supporting built-in tools/web search in this node.
- **Edge cases or potential failure types:**
  - Invalid API key
  - Model not available in the account/region
  - Web search tool may require specific OpenAI platform support
  - Token or rate-limit errors
- **Sub-workflow reference:** None

---

## 2.3 Block: Image Job Submission and Polling

### Overview
This block sends the prompt to kie.ai as an asynchronous generation job, pauses for 15 seconds, then polls the job status until it succeeds, fails, or times out. It includes branching logic and persistent retry counting.

### Nodes Involved
- Generate Infographic
- Polling Delay (15s)
- Check Job Status
- Route by Job Status
- Increment Retry
- Check Timeout

### Node Details

#### Generate Infographic
- **Type and technical role:** `n8n-nodes-base.httpRequest`; submits the image-generation task to kie.ai.
- **Configuration choices:**
  - Method: `POST`
  - URL: `https://api.kie.ai/api/v1/jobs/createTask`
  - Body sent as JSON
  - Authentication: generic credential type using HTTP Header Auth
  - `neverError` enabled so non-2xx responses stay in normal output instead of throwing
  - `onError: continueErrorOutput` enabled, allowing the workflow to continue even on request issues
  - Model submitted: `nano-banana-pro`
- **Key expressions or variables used:**
  - Prompt: `{{ JSON.stringify($json.output) }}`
  - Aspect ratio: `{{ $('On form submission').item.json['Aspect Ratio'] }}`
  - Resolution: `{{ $('On form submission').item.json.Resolution }}`
  - Output format: `{{ $('On form submission').item.json['Output Format'] }}`
- **Input and output connections:**
  - Input: `Build Image Prompt`
  - Main output 1: `Polling Delay (15s)`
  - Error output / alternate continuation: `Prepare Error Email`
- **Version-specific requirements:** Type version `4.3`; uses current HTTP Request node JSON-body and auth features.
- **Edge cases or potential failure types:**
  - Missing/invalid kie.ai auth header
  - API schema changes
  - Missing `output` property from agent node
  - Task creation may return malformed or unexpected JSON
- **Sub-workflow reference:** None

#### Polling Delay (15s)
- **Type and technical role:** `n8n-nodes-base.wait`; introduces a fixed delay between job creation or retry loops and the next status check.
- **Configuration choices:**
  - Wait amount: `15` seconds
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Input: from `Generate Infographic` or from retry branch via `Check Timeout`
  - Output: `Check Job Status`
- **Version-specific requirements:** Type version `1.1`
- **Edge cases or potential failure types:**
  - Workflow execution remains paused; environment must support resumed executions
  - Long wait chains can interact with instance execution limits
- **Sub-workflow reference:** None

#### Check Job Status
- **Type and technical role:** `n8n-nodes-base.httpRequest`; queries the status of the kie.ai task.
- **Configuration choices:**
  - URL: `https://api.kie.ai/api/v1/jobs/recordInfo`
  - Query parameter `taskId`
  - Authentication via the same HTTP Header Auth credential
  - `neverError` enabled
  - `onError: continueErrorOutput` enabled
- **Key expressions or variables used:**
  - `{{ $json.data?.taskId ?? $json.taskId }}`
- **Input and output connections:**
  - Input: `Polling Delay (15s)`
  - Main output 1: `Route by Job Status`
  - Error continuation output: `Prepare Error Email`
- **Version-specific requirements:** Type version `4.3`
- **Edge cases or potential failure types:**
  - Missing `taskId`
  - Invalid auth or expired token
  - API returns error payload without `data.state`
  - Network timeout or malformed response
- **Sub-workflow reference:** None

#### Route by Job Status
- **Type and technical role:** `n8n-nodes-base.switch`; branches execution based on `data.state`.
- **Configuration choices:**
  - Output `success` when `data.state == success`
  - Output `pending` when `data.state == waiting` or `generating`
  - Output `fail` when `data.state == fail`
  - Fallback output enabled and mapped to output index 2, effectively routing unmatched states to the failure/error path
- **Key expressions or variables used:**
  - `={{ $json.data.state }}`
- **Input and output connections:**
  - Input: `Check Job Status`
  - Output `success`: `Download Generated Image`
  - Output `pending`: `Increment Retry`
  - Output `fail` and fallback: `Prepare Error Email`
- **Version-specific requirements:** Type version `3.3`
- **Edge cases or potential failure types:**
  - Unexpected state values fall through to error handling
  - Missing `data.state` routes to fallback/error path
- **Sub-workflow reference:** None

#### Increment Retry
- **Type and technical role:** `n8n-nodes-base.code`; tracks how many polling cycles have occurred using workflow global static data.
- **Configuration choices:**
  - Uses `$getWorkflowStaticData('global')`
  - Resets counter automatically when execution ID changes
  - Increments `retryCount`
  - Carries forward `taskId`
- **Key expressions or variables used:**
  - `$getWorkflowStaticData('global')`
  - `$execution.id`
  - `$input.first().json.data?.taskId ?? $input.first().json.taskId`
- **Input and output connections:**
  - Input: `Route by Job Status` pending branch
  - Output: `Check Timeout`
- **Version-specific requirements:** Type version `2`
- **Edge cases or potential failure types:**
  - Static data persistence depends on n8n environment behavior
  - Concurrent executions are generally safe here because the code resets by execution ID, but global static data still deserves caution in clustered/high-concurrency deployments
  - Missing input may break code if response shape changes
- **Sub-workflow reference:** None

#### Check Timeout
- **Type and technical role:** `n8n-nodes-base.if`; stops polling after too many retries.
- **Configuration choices:**
  - Condition: `retryCount > 20`
  - True branch means timeout
  - False branch loops back to polling delay
- **Key expressions or variables used:**
  - `={{ $json.retryCount }}`
- **Input and output connections:**
  - Input: `Increment Retry`
  - True output: `Prepare Error Email`
  - False output: `Polling Delay (15s)`
- **Version-specific requirements:** Type version `2.2`
- **Edge cases or potential failure types:**
  - If retry count is missing or non-numeric, comparison may fail or behave unexpectedly
  - Total runtime can still be substantial due to wait and resume behavior
- **Sub-workflow reference:** None

---

## 2.4 Block: Success Path

### Overview
When the generation job succeeds, this block retrieves the image file from the URL returned by kie.ai and sends it to the configured recipient via Gmail.

### Nodes Involved
- Download Generated Image
- Email Image to User

### Node Details

#### Download Generated Image
- **Type and technical role:** `n8n-nodes-base.httpRequest`; downloads the generated image from the URL returned in the kie.ai result JSON.
- **Configuration choices:**
  - URL is dynamically extracted by parsing `data.resultJson`
  - `neverError` enabled
  - `onError: continueErrorOutput` enabled
- **Key expressions or variables used:**
  - `={{ JSON.parse($json.data.resultJson).resultUrls[0] }}`
- **Input and output connections:**
  - Input: `Route by Job Status` success branch
  - Main output 1: `Email Image to User`
  - Error continuation output: `Prepare Error Email`
- **Version-specific requirements:** Type version `4.3`
- **Edge cases or potential failure types:**
  - `resultJson` not valid JSON
  - `resultUrls` missing or empty
  - Download URL expired or inaccessible
  - Binary response handling depends on node defaults and n8n version; attachment delivery assumes the response is available as binary data
- **Sub-workflow reference:** None

#### Email Image to User
- **Type and technical role:** `n8n-nodes-base.gmail`; sends the generated infographic as an email attachment.
- **Configuration choices:**
  - Recipient is hard-coded as `user@example.com`
  - Subject: `New infographic generated`
  - Message: `Hi, the new infographic is attached to this email.`
  - Attribution disabled
  - Attachment configured from binary input
- **Key expressions or variables used:** No dynamic expressions in subject/body; attachment uses incoming binary implicitly.
- **Input and output connections:**
  - Input: `Download Generated Image`
  - Output: none
- **Version-specific requirements:** Type version `2.1`; requires Gmail OAuth2 credentials configured in n8n.
- **Edge cases or potential failure types:**
  - Gmail OAuth token invalid/expired
  - Recipient not updated from placeholder address
  - Missing binary attachment if the download step does not expose binary data correctly
- **Sub-workflow reference:** None

---

## 2.5 Block: Error Handling

### Overview
This block centralizes failure handling for job-creation errors, polling/API errors, explicit generation failures, timeout events, and image download failures. It builds a descriptive email using both the error context and the original form submission.

### Nodes Involved
- Prepare Error Email
- Send Error Email

### Node Details

#### Prepare Error Email
- **Type and technical role:** `n8n-nodes-base.code`; builds a subject/body pair for different failure classes.
- **Configuration choices:**
  - Reads current node input and original form submission
  - Supports three scenarios:
    1. `data.state === 'fail'` → generation failed
    2. `retryCount !== undefined` → timeout after polling limit
    3. all other cases → generic API error
  - Includes original request metadata in the email body
- **Key expressions or variables used:**
  - `$input.first().json`
  - `$('On form submission').item.json`
- **Input and output connections:**
  - Inputs can arrive from:
    - `Generate Infographic`
    - `Check Job Status`
    - `Route by Job Status`
    - `Download Generated Image`
    - `Check Timeout`
  - Output: `Send Error Email`
- **Version-specific requirements:** Type version `2`
- **Edge cases or potential failure types:**
  - Assumes `On form submission` data is available in execution context
  - Markdown-like formatting in plain email body may not render as rich text
  - Large raw error payloads may produce verbose email content
- **Sub-workflow reference:** None

#### Send Error Email
- **Type and technical role:** `n8n-nodes-base.gmail`; sends error notifications.
- **Configuration choices:**
  - Recipient hard-coded as `user@example.com`
  - Subject from expression
  - Message body from expression
  - Attribution disabled
- **Key expressions or variables used:**
  - Subject: `={{ $json.subject }}`
  - Message: `={{ $json.body }}`
- **Input and output connections:**
  - Input: `Prepare Error Email`
  - Output: none
- **Version-specific requirements:** Type version `2.1`
- **Edge cases or potential failure types:**
  - Gmail auth problems
  - Placeholder recipient not changed
  - If upstream code fails to return `subject` or `body`, email send may fail
- **Sub-workflow reference:** None

---

## 2.6 Block: Documentation and Visual Guidance

### Overview
These nodes do not affect execution logic. They document the workflow visually inside the n8n canvas and provide setup instructions and contextual grouping.

### Nodes Involved
- Overview
- Section: User Input
- Section: AI Prompt Engineering
- Section: Generate and Poll
- Section: Success Path
- Section: Error Handling

### Node Details

#### Overview
- **Type and technical role:** `n8n-nodes-base.stickyNote`; canvas documentation.
- **Configuration choices:** Contains the overall purpose, audience, flow summary, and setup instructions for OpenAI, kie.ai, and Gmail.
- **Input and output connections:** none
- **Version-specific requirements:** Type version `1`
- **Edge cases or potential failure types:** none
- **Sub-workflow reference:** None

#### Section: User Input
- **Type and technical role:** Sticky note for visual grouping.
- **Configuration choices:** Labels the input section as `① User Input`.
- **Input and output connections:** none
- **Version-specific requirements:** Type version `1`
- **Edge cases or potential failure types:** none
- **Sub-workflow reference:** None

#### Section: AI Prompt Engineering
- **Type and technical role:** Sticky note for visual grouping.
- **Configuration choices:** Explains GPT-4 with web search writes an optimized image prompt.
- **Input and output connections:** none
- **Version-specific requirements:** Type version `1`
- **Edge cases or potential failure types:** none
- **Sub-workflow reference:** None

#### Section: Generate and Poll
- **Type and technical role:** Sticky note for visual grouping.
- **Configuration choices:** Explains the kie.ai submission and 15-second polling with 20 retry cap.
- **Input and output connections:** none
- **Version-specific requirements:** Type version `1`
- **Edge cases or potential failure types:** none
- **Sub-workflow reference:** None

#### Section: Success Path
- **Type and technical role:** Sticky note for visual grouping.
- **Configuration choices:** Explains successful image download and email delivery.
- **Input and output connections:** none
- **Version-specific requirements:** Type version `1`
- **Edge cases or potential failure types:** none
- **Sub-workflow reference:** None

#### Section: Error Handling
- **Type and technical role:** Sticky note for visual grouping.
- **Configuration choices:** Explains handling of generation failure, timeout, and API errors with descriptive email output.
- **Input and output connections:** none
- **Version-specific requirements:** Type version `1`
- **Edge cases or potential failure types:** none
- **Sub-workflow reference:** None

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| On form submission | n8n-nodes-base.formTrigger | Entry point collecting infographic requirements from a web form |  | Build Image Prompt | ## ① User Input |
| Build Image Prompt | @n8n/n8n-nodes-langchain.agent | AI agent that researches and writes the final image-generation prompt | On form submission; Researcher (AI language model) | Generate Infographic | ## ② AI Prompt Engineering<br>GPT-4 with web search researches your topic and writes an optimized image generation prompt. |
| Researcher | @n8n/n8n-nodes-langchain.lmChatOpenAi | OpenAI model backend with web search used by the agent |  | Build Image Prompt | ## ② AI Prompt Engineering<br>GPT-4 with web search researches your topic and writes an optimized image generation prompt. |
| Generate Infographic | n8n-nodes-base.httpRequest | Submits async image-generation job to kie.ai | Build Image Prompt | Polling Delay (15s); Prepare Error Email | ## ③ Generate & Poll<br>Submits the job to kie.ai, then polls every 15s until complete, failed, or timed out (20 retries max). |
| Polling Delay (15s) | n8n-nodes-base.wait | Waits 15 seconds between status checks | Generate Infographic; Check Timeout | Check Job Status | ## ③ Generate & Poll<br>Submits the job to kie.ai, then polls every 15s until complete, failed, or timed out (20 retries max). |
| Check Job Status | n8n-nodes-base.httpRequest | Queries kie.ai for current task status | Polling Delay (15s) | Route by Job Status; Prepare Error Email | ## ③ Generate & Poll<br>Submits the job to kie.ai, then polls every 15s until complete, failed, or timed out (20 retries max). |
| Route by Job Status | n8n-nodes-base.switch | Branches execution based on task state | Check Job Status | Download Generated Image; Increment Retry; Prepare Error Email | ## ③ Generate & Poll<br>Submits the job to kie.ai, then polls every 15s until complete, failed, or timed out (20 retries max). |
| Increment Retry | n8n-nodes-base.code | Increments poll retry counter using workflow static data | Route by Job Status | Check Timeout | ## ③ Generate & Poll<br>Submits the job to kie.ai, then polls every 15s until complete, failed, or timed out (20 retries max).<br>## ⑤ Error Handling<br>Handles 3 failure scenarios: generation failure, polling timeout (>20 retries), and API errors. Sends a descriptive error email for each case. |
| Check Timeout | n8n-nodes-base.if | Stops polling after more than 20 retries | Increment Retry | Prepare Error Email; Polling Delay (15s) | ## ③ Generate & Poll<br>Submits the job to kie.ai, then polls every 15s until complete, failed, or timed out (20 retries max).<br>## ⑤ Error Handling<br>Handles 3 failure scenarios: generation failure, polling timeout (>20 retries), and API errors. Sends a descriptive error email for each case. |
| Download Generated Image | n8n-nodes-base.httpRequest | Downloads finished image from returned result URL | Route by Job Status | Email Image to User; Prepare Error Email | ## ④ Success<br>Downloads the generated image and emails it as an attachment. |
| Email Image to User | n8n-nodes-base.gmail | Sends generated infographic to recipient | Download Generated Image |  | ## ④ Success<br>Downloads the generated image and emails it as an attachment. |
| Prepare Error Email | n8n-nodes-base.code | Builds subject and body for failure notifications | Generate Infographic; Check Job Status; Route by Job Status; Download Generated Image; Check Timeout | Send Error Email | ## ⑤ Error Handling<br>Handles 3 failure scenarios: generation failure, polling timeout (>20 retries), and API errors. Sends a descriptive error email for each case. |
| Send Error Email | n8n-nodes-base.gmail | Sends failure or timeout notification email | Prepare Error Email |  | ## ⑤ Error Handling<br>Handles 3 failure scenarios: generation failure, polling timeout (>20 retries), and API errors. Sends a descriptive error email for each case. |
| Overview | n8n-nodes-base.stickyNote | Documentation note with purpose and setup instructions |  |  | ## 🖼️ AI Infographic Generator<br><br>**Generate research-backed, professionally designed infographics using GPT-4 web search and kie.ai's image AI - no design skills needed.**<br><br>---<br><br>### Who is this for<br>- Marketers & content creators who need data-driven visuals fast<br>- Educators and consultants producing professional infographics<br>- Anyone who wants to turn a topic into a polished visual without Canva or Photoshop<br><br>---<br><br>### How it works<br>1. User fills in a form (headline, topic, style, layout, colors, resolution, etc.)<br>2. An AI Agent (GPT-4 with web search) researches the topic and writes an optimized image generation prompt<br>3. The prompt is sent to **kie.ai** (nano-banana-pro model) to generate the infographic<br>4. The workflow polls for completion every 15 seconds (up to 20 retries / ~5 min)<br>5. On success: the image is downloaded and emailed as an attachment<br>6. On failure or timeout: a detailed error email is sent instead<br><br>---<br><br>### Setup<br>1. **OpenAI** - Add your OpenAI API key to the `Researcher` node credential<br>2. **kie.ai** - Add your kie.ai Bearer token to a `Header Auth` credential and connect it to `Generate Infographic` and `Check Job Status`<br>3. **Gmail** - Connect your Gmail account to `Email Image to User` and `Send Error Email`<br>4. Update the recipient (To) email address in both Gmail nodes<br>5. Activate the workflow and open the form URL |
| Section: User Input | n8n-nodes-base.stickyNote | Visual section label |  |  | ## ① User Input |
| Section: AI Prompt Engineering | n8n-nodes-base.stickyNote | Visual section label |  |  | ## ② AI Prompt Engineering<br>GPT-4 with web search researches your topic and writes an optimized image generation prompt. |
| Section: Generate and Poll | n8n-nodes-base.stickyNote | Visual section label |  |  | ## ③ Generate & Poll<br>Submits the job to kie.ai, then polls every 15s until complete, failed, or timed out (20 retries max). |
| Section: Success Path | n8n-nodes-base.stickyNote | Visual section label |  |  | ## ④ Success<br>Downloads the generated image and emails it as an attachment. |
| Section: Error Handling | n8n-nodes-base.stickyNote | Visual section label |  |  | ## ⑤ Error Handling<br>Handles 3 failure scenarios: generation failure, polling timeout (>20 retries), and API errors. Sends a descriptive error email for each case. |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it something like: **Generate Research-Backed Infographics with AI and kie.ai**.
   - Leave it inactive until credentials and recipient addresses are configured.

2. **Add the Form Trigger node**
   - Node type: **Form Trigger**
   - Name: **On form submission**
   - Set:
     - Form title: `Infographics Maker`
     - Form description: `Use AI to Generate Well-Researched, High-Quality Infographics`
     - Disable attribution/appended branding
   - Add these required fields exactly with these labels so downstream expressions work:
     1. `Main Headline` — textarea
     2. `Key Data Points` — textarea
     3. `Target Audience` — dropdown with: Corporate, Educational, Social Media, Technical
     4. `Tone of Voice` — dropdown with: Professional, Marketing, Neutral, Urgent, Casual
     5. `Visualization Options` — dropdown with: Prioritize Charts/Graphs, Use Icons & Illustrations, Incorporate Maps, Use Photo Realistic Elements
     6. `Art Style` — dropdown with: Flat Vector, Isometric, Corporate Memphis, Neon/Cyberpunk, Hand-Drawn/Sketch, Retro Pop Art
     7. `Color Palette` — text field
     8. `Layout Structure` — dropdown with:
        - Vertical flow from top to bottom
        - Horizontal linear flow from left to right
        - Comparison: Split screen (Left vs Right)
        - Central Hub / Mindmap
        - Step-by-Step: Numbered list layout
        - Modular Grid
     9. `Font Family` — dropdown with:
        - Modern Sans (Roboto/Helvetica)
        - Classic Serif (Garamond/Times)
        - Bold Headline (Oswald/Impact)
        - Tech/Code (Courier/Monospace)
        - Handwritten (Marker Style)
        - Elegant/Script (Cursive)
     10. `Information Density` — radio with: Minimalist, Balanced, Detailed
     11. `Aspect Ratio` — dropdown with: 9:16, 2:3, 3:4, 4:5, 1:1, 5:4, 4:3, 3:2, 16:9, 21:9
     12. `Resolution` — radio with: 1K, 2K, 4K
     13. `Output Format` — radio with: png, jpg

3. **Add the OpenAI chat model node**
   - Node type: **OpenAI Chat Model** from the LangChain/AI nodes
   - Name: **Researcher**
   - Choose model: `gpt-4.1-mini`
   - Enable built-in tool: **Web Search**
   - Set search context size to `medium`
   - Connect OpenAI credentials:
     - Credential type: **OpenAI API**
     - Use a valid API key with access to the chosen model and tool support

4. **Add the AI Agent node**
   - Node type: **AI Agent** from the LangChain/AI nodes
   - Name: **Build Image Prompt**
   - Set prompt mode to define the system/instruction text directly
   - Paste the full instruction prompt from the workflow logic:
     - role as art director/prompt engineer
     - use form inputs
     - use the `Researcher` model for internet-backed data gathering
     - output raw prompt only
   - Make sure the prompt references the exact form field labels:
     - `Main Headline`
     - `Key Data Points`
     - `Layout Structure`
     - `Visualization Options`
     - `Tone of Voice`
     - `Art Style`
     - `Information Density`
     - `Font Family`
     - `Color Palette`
   - Connect:
     - `On form submission` → `Build Image Prompt` (main)
     - `Researcher` → `Build Image Prompt` (AI language model)

5. **Add the HTTP Request node for job creation**
   - Node type: **HTTP Request**
   - Name: **Generate Infographic**
   - Method: `POST`
   - URL: `https://api.kie.ai/api/v1/jobs/createTask`
   - Authentication: **Generic Credential Type**
   - Generic auth type: **HTTP Header Auth**
   - Create and assign a credential such as:
     - Name: `kie.ai API`
     - Header likely using Bearer token authorization as required by kie.ai
   - Set body format to JSON
   - Enable send body
   - Enable response option `neverError`
   - Set node error handling to continue on error output
   - JSON body:
     - `model`: `nano-banana-pro`
     - `input.prompt`: the output from the AI Agent
     - `input.aspect_ratio`: from form trigger
     - `input.resolution`: from form trigger
     - `input.output_format`: from form trigger
   - Equivalent expression logic:
     - prompt = `{{$json.output}}` serialized as JSON string
     - aspect ratio = `$('On form submission').item.json['Aspect Ratio']`
     - resolution = `$('On form submission').item.json['Resolution']`
     - output format = `$('On form submission').item.json['Output Format']`
   - Connect `Build Image Prompt` → `Generate Infographic`

6. **Add a Wait node**
   - Node type: **Wait**
   - Name: **Polling Delay (15s)**
   - Set fixed wait amount to `15` seconds
   - Connect `Generate Infographic` main output → `Polling Delay (15s)`

7. **Add the HTTP Request node for polling**
   - Node type: **HTTP Request**
   - Name: **Check Job Status**
   - Method: `GET` or default query-based request compatible with n8n HTTP Request
   - URL: `https://api.kie.ai/api/v1/jobs/recordInfo`
   - Authentication: same **HTTP Header Auth** credential as above
   - Enable query parameters
   - Add:
     - `taskId` = `{{ $json.data?.taskId ?? $json.taskId }}`
   - Enable response option `neverError`
   - Set node error handling to continue on error output
   - Connect `Polling Delay (15s)` → `Check Job Status`

8. **Add the Switch node**
   - Node type: **Switch**
   - Name: **Route by Job Status**
   - Add outputs with renamed keys:
     - `success` when `{{$json.data.state}}` equals `success`
     - `pending` when `{{$json.data.state}}` equals `waiting` OR `generating`
     - `fail` when `{{$json.data.state}}` equals `fail`
   - Enable fallback output and route it to the error-handling branch
   - Connect `Check Job Status` → `Route by Job Status`

9. **Add the retry counter Code node**
   - Node type: **Code**
   - Name: **Increment Retry**
   - Use JavaScript
   - Logic required:
     - read workflow global static data
     - reset `retryCount` when execution ID changes
     - increment retry count
     - pass through `taskId`
   - Connect `Route by Job Status` pending output → `Increment Retry`

10. **Add the timeout IF node**
    - Node type: **If**
    - Name: **Check Timeout**
    - Condition:
      - numeric comparison
      - `{{$json.retryCount}} > 20`
    - Connect `Increment Retry` → `Check Timeout`

11. **Create the polling loop**
    - Connect:
      - `Check Timeout` false output → `Polling Delay (15s)`
    - This creates the loop for waiting and polling again.

12. **Add the success download node**
    - Node type: **HTTP Request**
    - Name: **Download Generated Image**
    - URL expression:
      - `{{ JSON.parse($json.data.resultJson).resultUrls[0] }}`
    - Enable `neverError`
    - Set error handling to continue on error output
    - Prefer response handling that preserves binary data for attachment sending
    - Connect `Route by Job Status` success output → `Download Generated Image`

13. **Add the Gmail node for successful delivery**
    - Node type: **Gmail**
    - Name: **Email Image to User**
    - Operation: send email
    - Gmail OAuth2 credential:
      - connect a valid Gmail account
    - Set:
      - To: replace `user@example.com` with the real recipient or dynamic recipient logic
      - Subject: `New infographic generated`
      - Message: `Hi, the new infographic is attached to this email.`
      - Disable appended attribution
    - Add attachment from incoming binary data
    - Connect `Download Generated Image` → `Email Image to User`

14. **Add the Code node for error message construction**
    - Node type: **Code**
    - Name: **Prepare Error Email**
    - Use JavaScript
    - Implement three cases:
      1. `data.state === 'fail'` → generation failed
      2. `retryCount !== undefined` → timeout after 20 polling attempts
      3. otherwise → generic API error
    - Pull original form values from `$('On form submission').item.json`
    - Return a JSON object with:
      - `subject`
      - `body`

15. **Add the Gmail node for error delivery**
    - Node type: **Gmail**
    - Name: **Send Error Email**
    - Operation: send email
    - Use the same Gmail OAuth2 credential
    - Set:
      - To: replace `user@example.com`
      - Subject: `={{ $json.subject }}`
      - Message: `={{ $json.body }}`
      - Disable appended attribution
    - Connect `Prepare Error Email` → `Send Error Email`

16. **Wire all error branches**
    - Connect the error/secondary outputs of these nodes to `Prepare Error Email`:
      - `Generate Infographic`
      - `Check Job Status`
      - `Download Generated Image`
    - Connect the `fail` branch of `Route by Job Status` to `Prepare Error Email`
    - Connect the `Check Timeout` true branch to `Prepare Error Email`
    - If fallback output is enabled in `Route by Job Status`, also direct that unmatched state path to `Prepare Error Email`

17. **Optional but recommended: add sticky notes**
    - Add canvas notes for:
      - overview/setup
      - user input
      - AI prompt engineering
      - generate & poll
      - success
      - error handling

18. **Configure credentials**
    - **OpenAI**
      - Add an OpenAI API credential to the `Researcher` node
    - **kie.ai**
      - Create an HTTP Header Auth credential
      - Configure authorization header per kie.ai API requirements
      - Attach it to both `Generate Infographic` and `Check Job Status`
    - **Gmail**
      - Create Gmail OAuth2 credential
      - Attach it to both Gmail nodes

19. **Test end-to-end**
    - Activate or test the form trigger
    - Submit a small request
    - Verify:
      - AI prompt is generated
      - kie.ai returns a task ID
      - polling transitions correctly
      - success sends an attachment
      - failure paths send meaningful error messages

20. **Production hardening recommendations**
    - Replace hard-coded email recipient with a form field if user-specific delivery is needed
    - Validate binary attachment handling in `Download Generated Image`
    - Consider storing task metadata externally if stronger retry tracking is needed
    - Consider adding a node to clear or isolate retry state if the workflow is heavily concurrent

**Sub-workflow setup:**  
This workflow does **not** invoke any sub-workflow node and has only one explicit entry point: the Form Trigger.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Generate research-backed, professionally designed infographics using GPT-4 web search and kie.ai's image AI - no design skills needed. | Workflow purpose |
| Marketers & content creators who need data-driven visuals fast. | Target users |
| Educators and consultants producing professional infographics. | Target users |
| Anyone who wants to turn a topic into a polished visual without Canva or Photoshop. | Target users |
| OpenAI - Add your OpenAI API key to the `Researcher` node credential. | Setup |
| kie.ai - Add your kie.ai Bearer token to a `Header Auth` credential and connect it to `Generate Infographic` and `Check Job Status`. | Setup |
| Gmail - Connect your Gmail account to `Email Image to User` and `Send Error Email`. | Setup |
| Update the recipient (To) email address in both Gmail nodes. | Setup |
| Activate the workflow and open the form URL. | Setup |