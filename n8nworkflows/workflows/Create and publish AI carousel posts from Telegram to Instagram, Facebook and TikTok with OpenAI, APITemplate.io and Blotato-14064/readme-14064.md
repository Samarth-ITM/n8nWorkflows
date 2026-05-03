Create and publish AI carousel posts from Telegram to Instagram, Facebook and TikTok with OpenAI, APITemplate.io and Blotato

https://n8nworkflows.xyz/workflows/create-and-publish-ai-carousel-posts-from-telegram-to-instagram--facebook-and-tiktok-with-openai--apitemplate-io-and-blotato-14064


# Create and publish AI carousel posts from Telegram to Instagram, Facebook and TikTok with OpenAI, APITemplate.io and Blotato

# 1. Workflow Overview

This workflow turns a **Telegram text message or voice note** into a reviewed, AI-generated **5-slide carousel post** and publishes it to **Instagram, Facebook, and TikTok**. It uses **OpenAI** for transcription and content generation, **Google Sheets** as a lightweight content log, **APITemplate.io** to render slide images, and **Blotato** to publish and monitor posts.

Typical use cases:
- Solo creators drafting quote-based carousel posts from ideas sent via Telegram
- Teams using Telegram as a lightweight approval channel
- Social media automation where one approved concept is repurposed across multiple platforms

## 1.1 Input Reception and Voice Handling
The workflow starts from a Telegram bot trigger. It extracts text from incoming messages and detects whether the user sent a text message or a voice note. If the input is a voice note, the audio file is fetched from Telegram and transcribed with OpenAI before continuing.

## 1.2 AI Drafting, Revision, and Approval Loop
The workflow sends the input text to an AI Agent backed by an OpenAI chat model, structured output parsing, and conversation memory keyed by Telegram user ID. The agent drafts quotes and a social caption, then waits for user feedback through Telegram until the content is explicitly approved.

## 1.3 Approved Content Preparation and Logging
Once approved, the workflow extracts the final quotes and caption, stores them in Google Sheets, and notifies the user that publishing preparation has started.

## 1.4 Carousel Image Generation
Each approved quote is injected into an APITemplate.io image template to create five separate carousel slides. The outputs are merged, and the workflow collects the generated public image URLs for publishing.

## 1.5 Social Publishing and Status Monitoring
The workflow creates one post each for Instagram, Facebook, and TikTok through Blotato. Each platform then enters its own status-check loop with waits, retries while the post is still processing, and Telegram notifications for success or failure.

---

# 2. Block-by-Block Analysis

## 2.1 Block: Telegram Input and Voice Transcription

**Overview:**  
This block receives inbound Telegram messages, normalizes text input, and routes voice notes into a transcription path. Its purpose is to ensure the downstream AI agent always receives usable text.

**Nodes Involved:**  
- Start: Telegram Message  
- Extract text from Telegram message  
- Check if input is a voice message  
- Get Voice File  
- Speech to Text

### Node: Start: Telegram Message
- **Type and role:** `n8n-nodes-base.telegramTrigger` — workflow entry point for incoming Telegram messages.
- **Configuration choices:** Listens to `message` updates only.
- **Key expressions or variables used:** None in the node itself; downstream nodes use `message.text`, `message.voice.file_id`, `message.chat.id`, and `message.from.id`.
- **Input and output connections:** Entry node → outputs to **Extract text from Telegram message**.
- **Version-specific requirements:** Type version `1`.
- **Edge cases / failures:**
  - Telegram credential/auth issues
  - Webhook registration problems
  - Bot not added or user not interacting correctly with the bot
  - Non-message updates are ignored by design
- **Sub-workflow reference:** None.

### Node: Extract text from Telegram message
- **Type and role:** `Set` node — creates a normalized `text` field from Telegram text content.
- **Configuration choices:** Sets `text = $json?.message?.text || ""`.
- **Key expressions or variables used:**  
  - `={{ $json?.message?.text || "" }}`
- **Input and output connections:** Input from **Start: Telegram Message** → output to **Check if input is a voice message**.
- **Version-specific requirements:** Type version `3.2`.
- **Edge cases / failures:**
  - If the message is not text, `text` becomes an empty string
  - If Telegram payload structure differs unexpectedly, expression could return empty output
- **Sub-workflow reference:** None.

### Node: Check if input is a voice message
- **Type and role:** `If` node — decides whether to transcribe audio or send text directly to the AI.
- **Configuration choices:** Tests whether `message.text` is empty.
- **Key expressions or variables used:**  
  - `={{ $json.message.text }}`
- **Routing behavior:**  
  - **True output:** text is empty → assumes voice note path
  - **False output:** text exists → goes directly to AI drafting
- **Input and output connections:** Input from **Extract text from Telegram message** → outputs to **Get Voice File** and **AI: Draft & Revise Post**.
- **Version-specific requirements:** Type version `2.2`.
- **Edge cases / failures:**
  - This logic assumes empty text means voice input; other Telegram message types like images/stickers would also satisfy the same condition
  - If a voice message payload is malformed or absent, downstream voice retrieval will fail
- **Sub-workflow reference:** None.

### Node: Get Voice File
- **Type and role:** `Telegram` node — downloads the Telegram file corresponding to the voice note.
- **Configuration choices:** Uses Telegram file resource with `fileId` from the trigger payload.
- **Key expressions or variables used:**  
  - `={{ $('Start: Telegram Message').item.json.message.voice.file_id }}`
- **Input and output connections:** Input from **Check if input is a voice message** (true branch) → output to **Speech to Text**.
- **Version-specific requirements:** Type version `1.1`.
- **Edge cases / failures:**
  - Missing `message.voice.file_id`
  - Telegram file no longer available
  - Bot permissions or credential issues
- **Sub-workflow reference:** None.

### Node: Speech to Text
- **Type and role:** `@n8n/n8n-nodes-langchain.openAi` — transcribes audio to text using OpenAI.
- **Configuration choices:** Resource `audio`, operation `transcribe`; otherwise default options.
- **Key expressions or variables used:** None shown directly; uses binary/file input from previous node.
- **Input and output connections:** Input from **Get Voice File** → output to **AI: Draft & Revise Post**.
- **Version-specific requirements:** Type version `1.3`; requires valid OpenAI credentials and compatible audio input.
- **Edge cases / failures:**
  - Unsupported or corrupted audio format
  - OpenAI quota/auth/rate-limit issues
  - Long audio causing timeouts or cost spikes
- **Sub-workflow reference:** None.

---

## 2.2 Block: AI Drafting, Review, Structured Parsing, and Approval

**Overview:**  
This block generates and refines social post content through an OpenAI-powered agent. It preserves short conversational history by Telegram user, enforces structured JSON output, and loops with Telegram review until explicit approval is detected.

**Nodes Involved:**  
- AI: Draft & Revise Post  
- OpenAI Chat Model  
- Window Buffer Memory  
- Structured Output Parser  
- Auto-fixing Output Parser  
- OpenAI Chat Model (parser)  
- Check if Approved  
- Send draft to Telegram for review

### Node: OpenAI Chat Model
- **Type and role:** `lmChatOpenAi` — primary LLM used by the AI agent.
- **Configuration choices:** Model set to `gpt-5.1`; response format is forced to `json_object`.
- **Key expressions or variables used:** None.
- **Input and output connections:** Connected to **AI: Draft & Revise Post** through the `ai_languageModel` port.
- **Version-specific requirements:** Type version `1.2`.
- **Edge cases / failures:**
  - Model availability changes
  - Invalid credential or insufficient OpenAI quota
  - Forced JSON output may still produce malformed content without parser recovery
- **Sub-workflow reference:** None.

### Node: Window Buffer Memory
- **Type and role:** LangChain memory node — preserves recent conversational context for iterative revisions.
- **Configuration choices:**  
  - Session key = Telegram sender ID  
  - Session type = custom key  
  - Context window length = 10
- **Key expressions or variables used:**  
  - `={{ $('Start: Telegram Message').first().json.message.from.id }}`
- **Input and output connections:** Connected to **AI: Draft & Revise Post** via `ai_memory`.
- **Version-specific requirements:** Type version `1.3`.
- **Edge cases / failures:**
  - If `message.from.id` is missing, sessions may collapse or fail
  - Memory is limited to the last 10 turns, so older revision context is discarded
- **Sub-workflow reference:** None.

### Node: Structured Output Parser
- **Type and role:** Structured output parser — defines the JSON schema expected from the AI pipeline.
- **Configuration choices:** Manual JSON schema with fields:
  - `approved` boolean
  - `quotes` array of strings, max 5
  - `followUpQuestion` string
  - `SocialMediaText` string
- **Key expressions or variables used:** None.
- **Input and output connections:** Outputs to **Auto-fixing Output Parser** via `ai_outputParser`.
- **Version-specific requirements:** Type version `1.3`.
- **Edge cases / failures:**
  - Schema mismatch if model returns unexpected keys or wrong types
  - Required fields are only `approved`, `quotes`, and `followUpQuestion`; `SocialMediaText` is described but not required
- **Sub-workflow reference:** None.

### Node: OpenAI Chat Model (parser)
- **Type and role:** Secondary OpenAI chat model used by the auto-fixing parser.
- **Configuration choices:** Model `gpt-5` with `json_schema` text formatting matching the approval schema.
- **Key expressions or variables used:** None.
- **Input and output connections:** Connected to **Auto-fixing Output Parser** via `ai_languageModel`.
- **Version-specific requirements:** Type version `1.3`.
- **Edge cases / failures:**
  - Same OpenAI credential/model availability concerns as the main model
  - Parser repair may still fail if upstream output is too inconsistent
- **Sub-workflow reference:** None.

### Node: Auto-fixing Output Parser
- **Type and role:** Auto-correction layer for malformed structured LLM output.
- **Configuration choices:** Default options.
- **Key expressions or variables used:** None.
- **Input and output connections:**  
  - Input from **Structured Output Parser**
  - LLM support from **OpenAI Chat Model (parser)**
  - Output to **AI: Draft & Revise Post**
- **Version-specific requirements:** Type version `1`.
- **Edge cases / failures:**
  - May increase latency and token usage
  - Cannot reliably fix logically incorrect but schema-valid output
- **Sub-workflow reference:** None.

### Node: AI: Draft & Revise Post
- **Type and role:** LangChain agent — core AI logic for drafting, revising, and approval-state output.
- **Configuration choices:**  
  - Prompt type: define manually
  - Input text: `={{ $json.text }}`
  - Has output parser: enabled
  - Extensive system prompt instructs the agent to:
    - Draft a short script
    - Produce 5 quotes
    - Produce one English social caption with hashtags
    - Keep all suggestions inside `followUpQuestion` until explicit approval
    - Output JSON only when structured output is required
    - On approval, return only final JSON
- **Key expressions or variables used:**  
  - `={{ $json.text }}`
- **Input and output connections:**  
  - Main input from **Check if input is a voice message** (text path) or **Speech to Text** (voice path)
  - LLM input from **OpenAI Chat Model**
  - Memory input from **Window Buffer Memory**
  - Parser input from **Auto-fixing Output Parser**
  - Main output to **Check if Approved**
- **Version-specific requirements:** Type version `3`.
- **Edge cases / failures:**
  - The prompt expects iterative approval behavior, but successful looping depends on how consistently the user replies
  - If input text is empty or transcription fails silently, the model may hallucinate or ask unclear follow-up questions
  - Structured outputs may still differ in casing; downstream nodes expect `output.SocialMediaText`
- **Sub-workflow reference:** None.

### Node: Check if Approved
- **Type and role:** `If` node — checks whether the AI output marks the content as approved.
- **Configuration choices:** Tests boolean truth of `output.approved`.
- **Key expressions or variables used:**  
  - `={{ $json.output.approved }}`
- **Routing behavior:**  
  - **True output:** proceed to content extraction and publishing prep
  - **False output:** send draft/revision back to Telegram
- **Input and output connections:** Input from **AI: Draft & Revise Post** → outputs to **Extract approved quotes** and **Send draft to Telegram for review**.
- **Version-specific requirements:** Type version `2.2`.
- **Edge cases / failures:**
  - If parser output is malformed or `approved` is missing, strict boolean evaluation may fail or route unexpectedly
- **Sub-workflow reference:** None.

### Node: Send draft to Telegram for review
- **Type and role:** `Telegram` node — sends the current AI draft back to the user.
- **Configuration choices:** Constructs a message by printing five quote slots, `SocialMediaText`, and `followUpQuestion`.
- **Key expressions or variables used:**  
  - References like `$('AI: Draft & Revise Post').item.json.output.quotes[0]`
  - `$('AI: Draft & Revise Post').item.json.output.SocialMediaText`
  - `$('AI: Draft & Revise Post').item.json.output.followUpQuestion`
  - Chat ID from trigger: `={{ $('Start: Telegram Message').first().json.message.chat.id }}`
- **Input and output connections:** Input from **Check if Approved** false branch. No downstream nodes.
- **Version-specific requirements:** Type version `1.2`.
- **Edge cases / failures:**
  - Before approval, quotes may intentionally be empty per prompt rules, so the message may contain blank lines
  - Telegram message length limits may be hit with verbose drafts
  - If AI output structure changes, expressions may break
- **Sub-workflow reference:** None.

---

## 2.3 Block: Approved Content Extraction and Logging

**Overview:**  
After approval, this block reshapes the final AI output into explicit fields, writes the result to Google Sheets, and informs the user that post generation has started.

**Nodes Involved:**  
- Extract approved quotes  
- Log approved quotes in Google Sheets  
- Notify user: preparing post

### Node: Extract approved quotes
- **Type and role:** `Set` node — flattens approved output fields into explicit properties for downstream nodes.
- **Configuration choices:** Creates:
  - `output.quotes[0]` to `output.quotes[4]`
  - `output.SocialMediaText`
- **Key expressions or variables used:** Direct references to `$json.output.quotes[n]` and `$json.output.SocialMediaText`.
- **Input and output connections:** Input from **Check if Approved** true branch → outputs to **Log approved quotes in Google Sheets** and **Notify user: preparing post**.
- **Version-specific requirements:** Type version `3.4`.
- **Edge cases / failures:**
  - If fewer than 5 quotes are returned, some fields become empty/undefined
  - Uses nested field names that may not map intuitively in all downstream contexts
- **Sub-workflow reference:** None.

### Node: Log approved quotes in Google Sheets
- **Type and role:** `Google Sheets` node — appends a row containing the approved content.
- **Configuration choices:**  
  - Operation: append
  - Spreadsheet ID placeholder: `YOUR_GOOGLE_SHEETS_ID`
  - Sheet/tab placeholder: `YOUR_GSHEET_TAB_GID`
  - Maps:
    - `run_id = $now.toISO()`
    - `quote1`..`quote5` from approved quotes
    - `social_media_text` from `output.SocialMediaText`
  - Removes wrapping straight quotes using `.replace(/^\"|\"$/g, '')`
- **Key expressions or variables used:**  
  - `={{ $json.output.quotes[0].replace(/^\"|\"$/g, '') }}`
  - similar for all quotes
  - `={{ $now.toISO() }}`
  - `={{ $json.output.SocialMediaText.replace(/^\"|\"$/g, '') }}`
- **Input and output connections:** Input from **Extract approved quotes** → outputs in parallel to all five APITemplate slide nodes.
- **Version-specific requirements:** Type version `4.7`; requires Google Sheets OAuth2 credentials.
- **Edge cases / failures:**
  - Placeholder spreadsheet identifiers must be replaced
  - `.replace(...)` will fail if a quote or caption is `null`/`undefined`
  - Google API permission issues or tab mismatch
- **Sub-workflow reference:** None.

### Node: Notify user: preparing post
- **Type and role:** `Telegram` node — informs the user that generation/publishing is underway.
- **Configuration choices:** Static text: `Your post is being prepared.`
- **Key expressions or variables used:**  
  - Chat ID: `={{ $('Start: Telegram Message').first().json.message.chat.id }}`
- **Input and output connections:** Input from **Extract approved quotes**. No downstream nodes.
- **Version-specific requirements:** Type version `1.2`.
- **Edge cases / failures:**
  - Telegram auth or delivery issues
- **Sub-workflow reference:** None.

---

## 2.4 Block: Carousel Image Generation

**Overview:**  
This block generates one image per approved quote using APITemplate.io, merges the five generation branches, and assembles the final media URL bundle and caption for cross-platform posting.

**Nodes Involved:**  
- Generate carousel slide 1  
- Generate carousel slide 2  
- Generate carousel slide 3  
- Generate carousel slide 4  
- Generate carousel slide 5  
- Merge all carousel slides  
- Collect all image URLs for publishing

### Node: Generate carousel slide 1
- **Type and role:** `apiTemplateIo` — renders slide 1 image from a template.
- **Configuration choices:**  
  - Download enabled
  - Template ID placeholder: `YOUR_APITEMPLATE_TEMPLATE_ID`
  - Overrides text layer `text_1_1_1.text` with `quote1`
- **Key expressions or variables used:**  
  - `={{ $json.quote1 }}`
- **Input and output connections:** Input from **Log approved quotes in Google Sheets** → output to **Merge all carousel slides** input 0.
- **Version-specific requirements:** Type version `1`.
- **Edge cases / failures:**
  - This node references `quote1` at top level, but the Google Sheets output may not expose data exactly as expected depending on node response format
  - Invalid template ID or override key mismatch
- **Sub-workflow reference:** None.

### Node: Generate carousel slide 2
- **Type and role:** `apiTemplateIo` — renders slide 2.
- **Configuration choices:** Overrides `text_1_1_1.text` with quote 2 from a node named **Append row in sheet**.
- **Key expressions or variables used:**  
  - `={{ $('Append row in sheet').item.json.quote2 }}`
- **Input and output connections:** Input from **Log approved quotes in Google Sheets** → output to **Merge all carousel slides** input 1.
- **Version-specific requirements:** Type version `1`.
- **Edge cases / failures:**
  - **Important mismatch:** there is no node named **Append row in sheet** in this workflow. The actual node name is **Log approved quotes in Google Sheets**. This expression will fail unless renamed or corrected.
  - Invalid template setup or credential issues
- **Sub-workflow reference:** None.

### Node: Generate carousel slide 3
- **Type and role:** `apiTemplateIo` — renders slide 3.
- **Configuration choices:** Same pattern as slide 2 for quote 3.
- **Key expressions or variables used:**  
  - `={{ $('Append row in sheet').item.json.quote3 }}`
- **Input and output connections:** Input from **Log approved quotes in Google Sheets** → output to **Merge all carousel slides** input 2.
- **Version-specific requirements:** Type version `1`.
- **Edge cases / failures:**
  - Same broken node reference to **Append row in sheet**
- **Sub-workflow reference:** None.

### Node: Generate carousel slide 4
- **Type and role:** `apiTemplateIo` — renders slide 4.
- **Configuration choices:** Same pattern as slide 2 for quote 4.
- **Key expressions or variables used:**  
  - `={{ $('Append row in sheet').item.json.quote4 }}`
- **Input and output connections:** Input from **Log approved quotes in Google Sheets** → output to **Merge all carousel slides** input 3.
- **Version-specific requirements:** Type version `1`.
- **Edge cases / failures:**
  - Same broken node reference to **Append row in sheet**
- **Sub-workflow reference:** None.

### Node: Generate carousel slide 5
- **Type and role:** `apiTemplateIo` — renders slide 5.
- **Configuration choices:** Same pattern as slide 2 for quote 5.
- **Key expressions or variables used:**  
  - `={{ $('Append row in sheet').item.json.quote5 }}`
- **Input and output connections:** Input from **Log approved quotes in Google Sheets** → output to **Merge all carousel slides** input 4.
- **Version-specific requirements:** Type version `1`.
- **Edge cases / failures:**
  - Same broken node reference to **Append row in sheet**
- **Sub-workflow reference:** None.

### Node: Merge all carousel slides
- **Type and role:** `Merge` node — waits for all five slide-generation branches.
- **Configuration choices:** `numberInputs = 5`.
- **Key expressions or variables used:** None.
- **Input and output connections:** Inputs from all five slide nodes → output to **Collect all image URLs for publishing**.
- **Version-specific requirements:** Type version `3.2`.
- **Edge cases / failures:**
  - If any one slide generation fails, merge never completes
- **Sub-workflow reference:** None.

### Node: Collect all image URLs for publishing
- **Type and role:** `Set` node — gathers all generated `download_url` values and the social caption into one item.
- **Configuration choices:**  
  - `executeOnce = true`
  - retries enabled (`maxTries: 5`, 5-second delay)
  - Assigns:
    - `img_url_1` from node **Create an image1**
    - `img_url_2` from node **Create an image2**
    - `img_url_3` from node **Create an image3**
    - `img_url_4` from node **Create an image4**
    - `img_url_5` from node **Create an image5**
    - `post_ext` from node **Append row in sheet**
- **Key expressions or variables used:**  
  - `={{ $('Create an image1').first().json.download_url }}`
  - `={{ $('Create an image2').first().json.download_url }}`
  - `={{ $('Create an image3').first().json.download_url }}`
  - `={{ $('Create an image4').first().json.download_url }}`
  - `={{ $('Create an image5').first().json.download_url }}`
  - `={{ $('Append row in sheet').item.json.social_media_text }}`
- **Input and output connections:** Input from **Merge all carousel slides** → outputs to **Create Instagram Post**, **Create Facebook Post**, and **Create TikTok Post**.
- **Version-specific requirements:** Type version `3.4`.
- **Edge cases / failures:**
  - **Critical naming mismatch:** none of the referenced nodes (**Create an image1..5**, **Append row in sheet**) exist in this workflow
  - One field is named `=img_url_3` instead of `img_url_3`, which would break later expressions expecting `img_url_3`
  - Because publishing nodes use `$json.img_url_3`, TikTok/Facebook/Instagram media URL lists will be incomplete or invalid unless fixed
- **Sub-workflow reference:** None.

---

## 2.5 Block: Instagram Publishing and Monitoring

**Overview:**  
This block submits the carousel to Instagram via Blotato, waits, polls post status, retries while processing, and notifies the Telegram user of success or failure.

**Nodes Involved:**  
- Create Instagram Post  
- Wait 25s before checking Instagram  
- Instagram Check Post Status  
- Instagram post published?  
- Instagram still processing?  
- Retry: wait 5s for Instagram  
- Send Instagram success notification  
- Send Instagram error notification

### Node: Create Instagram Post
- **Type and role:** `@blotato/n8n-nodes-blotato.blotato` — creates an Instagram post submission.
- **Configuration choices:**  
  - Account ID placeholder: `YOUR_INSTAGRAM_ACCOUNT_ID`
  - Caption/body from `post_ext`
  - Media URLs from `img_url_1`..`img_url_5`, comma-separated
- **Key expressions or variables used:**  
  - `={{ $json.post_ext }}`
  - `={{ $json.img_url_1 }},{{ $json.img_url_2 }},{{ $json.img_url_3 }},{{ $json.img_url_4 }},{{ $json.img_url_5 }}`
- **Input and output connections:** Input from **Collect all image URLs for publishing** → output to **Wait 25s before checking Instagram**.
- **Version-specific requirements:** Type version `2`; requires community node `@blotato/n8n-nodes-blotato`, typically self-hosted n8n.
- **Edge cases / failures:**
  - Missing/invalid Blotato credentials
  - Invalid account ID
  - Broken `img_url_3` due to upstream field naming issue
  - Media URL accessibility problems
- **Sub-workflow reference:** None.

### Node: Wait 25s before checking Instagram
- **Type and role:** `Wait` node — provides initial delay before first status poll.
- **Configuration choices:** Wait amount `25`.
- **Key expressions or variables used:** None.
- **Input and output connections:** Input from **Create Instagram Post** → output to **Instagram Check Post Status**.
- **Version-specific requirements:** Type version `1.1`.
- **Edge cases / failures:**
  - Wait duration may be too short or too long depending on platform processing speed
- **Sub-workflow reference:** None.

### Node: Instagram Check Post Status
- **Type and role:** Blotato node — fetches post submission status.
- **Configuration choices:** Operation `get`; post submission ID from Instagram post creation node.
- **Key expressions or variables used:**  
  - `={{ $('Create Instagram Post').item.json.postSubmissionId }}`
- **Input and output connections:** Input from **Wait 25s before checking Instagram** and retry loop → output to **Instagram post published?**
- **Version-specific requirements:** Type version `2`.
- **Edge cases / failures:**
  - Missing `postSubmissionId`
  - API failures or delayed consistency
- **Sub-workflow reference:** None.

### Node: Instagram post published?
- **Type and role:** `If` node — checks whether Blotato reports `published`.
- **Configuration choices:** Compares `$json.status` to `published`.
- **Key expressions or variables used:**  
  - `={{ $json.status }}`
- **Routing behavior:**  
  - **True:** success notification
  - **False:** continue to processing check
- **Input and output connections:** Input from **Instagram Check Post Status** → outputs to **Send Instagram success notification** and **Instagram still processing?**
- **Version-specific requirements:** Type version `2.2`.
- **Edge cases / failures:**
  - Unexpected statuses such as `failed`, `rejected`, or null are all treated as not published
- **Sub-workflow reference:** None.

### Node: Instagram still processing?
- **Type and role:** `If` node — checks whether status is still `in-progress`.
- **Configuration choices:** Compares `$json.status` to `in-progress`.
- **Key expressions or variables used:**  
  - `={{ $json.status }}`
- **Routing behavior:**  
  - **True:** retry loop
  - **False:** send error notification
- **Input and output connections:** Input from **Instagram post published?** false branch → outputs to **Retry: wait 5s for Instagram** and **Send Instagram error notification**.
- **Version-specific requirements:** Type version `2.2`.
- **Edge cases / failures:**
  - Any non-`in-progress` failure state is treated uniformly as an error
- **Sub-workflow reference:** None.

### Node: Retry: wait 5s for Instagram
- **Type and role:** `Wait` node — delay between status rechecks.
- **Configuration choices:** Default wait configuration; no explicit amount shown.
- **Key expressions or variables used:** None.
- **Input and output connections:** Input from **Instagram still processing?** true branch → output to **Instagram Check Post Status**.
- **Version-specific requirements:** Type version `1.1`.
- **Edge cases / failures:**
  - With no explicit amount configured, behavior depends on n8n defaults; this should be verified manually
  - No max retry counter is implemented, so this can loop indefinitely while status remains `in-progress`
- **Sub-workflow reference:** None.

### Node: Send Instagram success notification
- **Type and role:** `Telegram` node — informs the user that publishing succeeded.
- **Configuration choices:** Sends public URL in the message body.
- **Key expressions or variables used:**  
  - `=Post successfully published: \n{{ $json.publicUrl }}`
  - chat ID from trigger
- **Input and output connections:** Input from **Instagram post published?** true branch. No downstream nodes.
- **Version-specific requirements:** Type version `1.2`.
- **Edge cases / failures:**
  - `publicUrl` may be missing even if status is published
- **Sub-workflow reference:** None.

### Node: Send Instagram error notification
- **Type and role:** `Telegram` node — notifies the user of upload failure.
- **Configuration choices:** Static error text.
- **Key expressions or variables used:**  
  - Chat ID from trigger: `={{ $('Start: Telegram Message').item.json.message.chat.id }}`
- **Input and output connections:** Input from **Instagram still processing?** false branch. No downstream nodes.
- **Version-specific requirements:** Type version `1.2`.
- **Edge cases / failures:**
  - Does not include error details from Blotato, so diagnosis requires execution logs
- **Sub-workflow reference:** None.

---

## 2.6 Block: Facebook Publishing and Monitoring

**Overview:**  
This block mirrors the Instagram flow for Facebook page publishing. It submits the carousel, waits, polls status, retries while still processing, and notifies the user of the outcome.

**Nodes Involved:**  
- Create Facebook Post  
- Wait 10s before checking Facebook  
- Facebook Check Post Status  
- Facebook post published?  
- Facebook still processing?  
- Retry: wait 5s for Facebook  
- Send Facebook error notification

### Node: Create Facebook Post
- **Type and role:** Blotato node — creates a Facebook post submission.
- **Configuration choices:**  
  - Platform set to `facebook`
  - Account ID placeholder and page ID placeholder
  - Caption from `post_ext`
  - Media URLs from `img_url_1`..`img_url_5`
- **Key expressions or variables used:**  
  - `={{ $json.post_ext }}`
  - comma-separated image URL expression
- **Input and output connections:** Input from **Collect all image URLs for publishing** → output to **Wait 10s before checking Facebook**.
- **Version-specific requirements:** Type version `2`.
- **Edge cases / failures:**
  - Invalid account/page IDs
  - Broken upstream URL fields
- **Sub-workflow reference:** None.

### Node: Wait 10s before checking Facebook
- **Type and role:** `Wait` node — initial delay before first Facebook poll.
- **Configuration choices:** Wait amount `10`.
- **Input and output connections:** Input from **Create Facebook Post** → output to **Facebook Check Post Status**.
- **Version-specific requirements:** Type version `1.1`.
- **Edge cases / failures:** Delay may be insufficient for some submissions.

### Node: Facebook Check Post Status
- **Type and role:** Blotato node — fetches Facebook submission status.
- **Configuration choices:** Operation `get`; post submission ID from **Create Facebook Post**.
- **Key expressions or variables used:**  
  - `={{ $('Create Facebook Post').item.json.postSubmissionId }}`
- **Input and output connections:** Input from initial wait and retry wait → output to **Facebook post published?**
- **Version-specific requirements:** Type version `2`.
- **Edge cases / failures:** Missing ID, API failures, stale status.

### Node: Facebook post published?
- **Type and role:** `If` node — checks for `published` status.
- **Configuration choices:** `$json.status == "published"`.
- **Input and output connections:** Input from **Facebook Check Post Status** → false branch to **Facebook still processing?**; true branch ends with no notification node connected.
- **Version-specific requirements:** Type version `2.2`.
- **Edge cases / failures:**
  - **Design gap:** unlike Instagram and TikTok, there is no Facebook success notification node connected
- **Sub-workflow reference:** None.

### Node: Facebook still processing?
- **Type and role:** `If` node — checks whether status is `in-progress`.
- **Configuration choices:** `$json.status == "in-progress"`.
- **Input and output connections:** Input from **Facebook post published?** false branch → true to **Retry: wait 5s for Facebook**, false to **Send Facebook error notification**.
- **Version-specific requirements:** Type version `2.2`.
- **Edge cases / failures:** All non-published and non-in-progress states become generic errors.

### Node: Retry: wait 5s for Facebook
- **Type and role:** `Wait` node — retry delay.
- **Configuration choices:** No explicit amount visible.
- **Input and output connections:** Input from **Facebook still processing?** true branch → output to **Facebook Check Post Status**.
- **Version-specific requirements:** Type version `1.1`.
- **Edge cases / failures:**
  - Potential infinite loop due to no retry cap
  - Default wait duration should be verified

### Node: Send Facebook error notification
- **Type and role:** `Telegram` node — sends failure message.
- **Configuration choices:** Static text says Instagram instead of Facebook.
- **Key expressions or variables used:** Chat ID from trigger.
- **Input and output connections:** Input from **Facebook still processing?** false branch.
- **Version-specific requirements:** Type version `1.2`.
- **Edge cases / failures:**
  - **Content bug:** message text is incorrect: `"Error while uploading your post to Instagram"`
- **Sub-workflow reference:** None.

---

## 2.7 Block: TikTok Publishing and Monitoring

**Overview:**  
This block publishes the carousel to TikTok via Blotato and monitors submission status. It includes TikTok-specific publishing options such as auto music, privacy, and AI-generated content marking.

**Nodes Involved:**  
- Create TikTok Post  
- Wait 10s before checking TikTok  
- TikTok Check Post Status  
- TikTok post published?  
- TikTok still processing?  
- Retry: wait 5s for TikTok  
- Send TikTok error notification

### Node: Create TikTok Post
- **Type and role:** Blotato node — creates a TikTok post submission.
- **Configuration choices:**  
  - Platform `tiktok`
  - Account ID placeholder
  - Caption from `post_ext`
  - Media URLs from `img_url_1`..`img_url_5`
  - Title: `Follow for more`
  - Auto-add music: enabled
  - Privacy level: `SELF_ONLY`
  - AI generated flag: true
- **Key expressions or variables used:**  
  - `={{ $json.post_ext }}`
  - comma-separated media URLs
- **Input and output connections:** Input from **Collect all image URLs for publishing** → output to **Wait 10s before checking TikTok**.
- **Version-specific requirements:** Type version `2`.
- **Edge cases / failures:**
  - `SELF_ONLY` makes the post non-public/private, which may not be intended in production
  - Broken `img_url_3` issue also applies here
- **Sub-workflow reference:** None.

### Node: Wait 10s before checking TikTok
- **Type and role:** `Wait` node — initial delay before polling.
- **Configuration choices:** Wait amount `10`.
- **Input and output connections:** Input from **Create TikTok Post** → output to **TikTok Check Post Status**.
- **Version-specific requirements:** Type version `1.1`.

### Node: TikTok Check Post Status
- **Type and role:** Blotato node — retrieves TikTok submission status.
- **Configuration choices:** Operation `get`; post submission ID from **Create TikTok Post**.
- **Key expressions or variables used:**  
  - `={{ $('Create TikTok Post').item.json.postSubmissionId }}`
- **Input and output connections:** Input from initial wait and retry wait → output to **TikTok post published?**
- **Version-specific requirements:** Type version `2`.

### Node: TikTok post published?
- **Type and role:** `If` node — checks for `published`.
- **Configuration choices:** `$json.status == "published"`.
- **Input and output connections:** Input from **TikTok Check Post Status** → false branch to **TikTok still processing?**; true branch ends without success notification.
- **Version-specific requirements:** Type version `2.2`.
- **Edge cases / failures:**
  - **Design gap:** no TikTok success notification is connected, despite the sticky note implying confirmation or error
- **Sub-workflow reference:** None.

### Node: TikTok still processing?
- **Type and role:** `If` node — checks for `in-progress`.
- **Configuration choices:** `$json.status == "in-progress"`.
- **Input and output connections:** Input from **TikTok post published?** false branch → true to **Retry: wait 5s for TikTok**, false to **Send TikTok error notification**.
- **Version-specific requirements:** Type version `2.2`.

### Node: Retry: wait 5s for TikTok
- **Type and role:** `Wait` node — delay before status retry.
- **Configuration choices:** No explicit amount visible.
- **Input and output connections:** Input from **TikTok still processing?** true branch → output to **TikTok Check Post Status**.
- **Version-specific requirements:** Type version `1.1`.
- **Edge cases / failures:**
  - Potential infinite retry loop
  - Default timing should be verified

### Node: Send TikTok error notification
- **Type and role:** `Telegram` node — notifies on TikTok publishing failure.
- **Configuration choices:** Static text incorrectly refers to Instagram.
- **Key expressions or variables used:** Chat ID from trigger.
- **Input and output connections:** Input from **TikTok still processing?** false branch.
- **Version-specific requirements:** Type version `1.2`.
- **Edge cases / failures:**
  - **Content bug:** message says Instagram instead of TikTok
- **Sub-workflow reference:** None.

---

## 2.8 Block: Documentation Sticky Notes

**Overview:**  
These nodes are purely informational and document the workflow purpose, setup, and visual grouping. They do not affect execution but are important for maintainability and context.

**Nodes Involved:**  
- Sticky Note  
- Sticky Note1  
- Sticky Note2  
- Sticky Note3  
- Sticky Note4  
- Sticky Note5  
- Sticky Note6

### Node: Sticky Note
- **Type and role:** `stickyNote` — high-level workflow description and setup instructions.
- **Configuration choices:** Large note covering the left-side overview.
- **Key content:** Explains the end-to-end flow and required credentials/services, including the need for the Blotato community node.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases / failures:** None; non-executable.
- **Sub-workflow reference:** None.

### Node: Sticky Note1
- **Type and role:** `stickyNote` — labels Step 1 input/transcription section.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.

### Node: Sticky Note2
- **Type and role:** `stickyNote` — labels Step 2 AI drafting/approval section.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.

### Node: Sticky Note3
- **Type and role:** `stickyNote` — labels Step 3 image generation section.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.

### Node: Sticky Note4
- **Type and role:** `stickyNote` — labels Step 4.a Instagram publishing section.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.

### Node: Sticky Note5
- **Type and role:** `stickyNote` — labels Step 4.b Facebook publishing section.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.

### Node: Sticky Note6
- **Type and role:** `stickyNote` — labels Step 4.c TikTok publishing section.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Start: Telegram Message | telegramTrigger | Entry point for Telegram messages |  | Extract text from Telegram message | ## Create and publish AI carousel posts to Instagram, Facebook, and TikTok<br>Turn a **Telegram message or voice note** into polished carousel posts published simultaneously to **Instagram, Facebook, and TikTok** via Blotato.<br><br>## How it works<br><br>1. Send a topic to the Telegram bot (text or voice note).<br>2. Voice notes are transcribed via OpenAI Whisper.<br>3. An AI Agent (GPT with conversation memory) drafts a script, 5 carousel quotes, and a social media caption with hashtags.<br>4. You review and iterate via Telegram chat until you approve.<br>5. Approved quotes are logged in Google Sheets.<br>6. APITemplate.io generates 5 styled carousel slide images from your template.<br>7. The carousel is published to Instagram, Facebook, and TikTok via Blotato.<br>8. Each platform's publish status is monitored -- you get a Telegram confirmation or error.<br><br>## Setup steps<br><br>1. **Telegram Bot** -- Create via @BotFather, add the API token as credential.<br>2. **OpenAI** -- Add your API key (used for the AI Agent and Whisper transcription).<br>3. **Google Sheets** -- Create OAuth2 credentials. Set up a spreadsheet with columns: run_id, quote1-5, social_media_text. Update the Sheet ID in the Google Sheets node.<br>4. **APITemplate.io** -- Create an account, design a carousel slide template, and set the template ID in the image generation nodes.<br>5. **Blotato** -- Connect your Instagram, Facebook, and TikTok accounts. Add the Blotato API credential and update account/page IDs in the publishing nodes.<br><br>> **Community node required:** `@blotato/n8n-nodes-blotato` -- this template works on **self-hosted n8n only**.<br>### Step 1: Telegram Input & Voice Transcription<br>User sends a text message or voice note. Voice notes are transcribed via OpenAI Whisper before being passed to the AI agent. |
| Extract text from Telegram message | set | Normalizes Telegram text into `text` field | Start: Telegram Message | Check if input is a voice message | ## Create and publish AI carousel posts to Instagram, Facebook, and TikTok<br>Turn a **Telegram message or voice note** into polished carousel posts published simultaneously to **Instagram, Facebook, and TikTok** via Blotato.<br><br>## How it works<br><br>1. Send a topic to the Telegram bot (text or voice note).<br>2. Voice notes are transcribed via OpenAI Whisper.<br>3. An AI Agent (GPT with conversation memory) drafts a script, 5 carousel quotes, and a social media caption with hashtags.<br>4. You review and iterate via Telegram chat until you approve.<br>5. Approved quotes are logged in Google Sheets.<br>6. APITemplate.io generates 5 styled carousel slide images from your template.<br>7. The carousel is published to Instagram, Facebook, and TikTok via Blotato.<br>8. Each platform's publish status is monitored -- you get a Telegram confirmation or error.<br><br>## Setup steps<br><br>1. **Telegram Bot** -- Create via @BotFather, add the API token as credential.<br>2. **OpenAI** -- Add your API key (used for the AI Agent and Whisper transcription).<br>3. **Google Sheets** -- Create OAuth2 credentials. Set up a spreadsheet with columns: run_id, quote1-5, social_media_text. Update the Sheet ID in the Google Sheets node.<br>4. **APITemplate.io** -- Create an account, design a carousel slide template, and set the template ID in the image generation nodes.<br>5. **Blotato** -- Connect your Instagram, Facebook, and TikTok accounts. Add the Blotato API credential and update account/page IDs in the publishing nodes.<br><br>> **Community node required:** `@blotato/n8n-nodes-blotato` -- this template works on **self-hosted n8n only**.<br>### Step 1: Telegram Input & Voice Transcription<br>User sends a text message or voice note. Voice notes are transcribed via OpenAI Whisper before being passed to the AI agent. |
| Check if input is a voice message | if | Routes text directly or voice to transcription | Extract text from Telegram message | Get Voice File; AI: Draft & Revise Post | ## Create and publish AI carousel posts to Instagram, Facebook, and TikTok<br>Turn a **Telegram message or voice note** into polished carousel posts published simultaneously to **Instagram, Facebook, and TikTok** via Blotato.<br><br>## How it works<br><br>1. Send a topic to the Telegram bot (text or voice note).<br>2. Voice notes are transcribed via OpenAI Whisper.<br>3. An AI Agent (GPT with conversation memory) drafts a script, 5 carousel quotes, and a social media caption with hashtags.<br>4. You review and iterate via Telegram chat until you approve.<br>5. Approved quotes are logged in Google Sheets.<br>6. APITemplate.io generates 5 styled carousel slide images from your template.<br>7. The carousel is published to Instagram, Facebook, and TikTok via Blotato.<br>8. Each platform's publish status is monitored -- you get a Telegram confirmation or error.<br><br>## Setup steps<br><br>1. **Telegram Bot** -- Create via @BotFather, add the API token as credential.<br>2. **OpenAI** -- Add your API key (used for the AI Agent and Whisper transcription).<br>3. **Google Sheets** -- Create OAuth2 credentials. Set up a spreadsheet with columns: run_id, quote1-5, social_media_text. Update the Sheet ID in the Google Sheets node.<br>4. **APITemplate.io** -- Create an account, design a carousel slide template, and set the template ID in the image generation nodes.<br>5. **Blotato** -- Connect your Instagram, Facebook, and TikTok accounts. Add the Blotato API credential and update account/page IDs in the publishing nodes.<br><br>> **Community node required:** `@blotato/n8n-nodes-blotato` -- this template works on **self-hosted n8n only**.<br>### Step 1: Telegram Input & Voice Transcription<br>User sends a text message or voice note. Voice notes are transcribed via OpenAI Whisper before being passed to the AI agent. |
| Get Voice File | telegram | Downloads Telegram voice file | Check if input is a voice message | Speech to Text | ## Create and publish AI carousel posts to Instagram, Facebook, and TikTok<br>Turn a **Telegram message or voice note** into polished carousel posts published simultaneously to **Instagram, Facebook, and TikTok** via Blotato.<br><br>## How it works<br><br>1. Send a topic to the Telegram bot (text or voice note).<br>2. Voice notes are transcribed via OpenAI Whisper.<br>3. An AI Agent (GPT with conversation memory) drafts a script, 5 carousel quotes, and a social media caption with hashtags.<br>4. You review and iterate via Telegram chat until you approve.<br>5. Approved quotes are logged in Google Sheets.<br>6. APITemplate.io generates 5 styled carousel slide images from your template.<br>7. The carousel is published to Instagram, Facebook, and TikTok via Blotato.<br>8. Each platform's publish status is monitored -- you get a Telegram confirmation or error.<br><br>## Setup steps<br><br>1. **Telegram Bot** -- Create via @BotFather, add the API token as credential.<br>2. **OpenAI** -- Add your API key (used for the AI Agent and Whisper transcription).<br>3. **Google Sheets** -- Create OAuth2 credentials. Set up a spreadsheet with columns: run_id, quote1-5, social_media_text. Update the Sheet ID in the Google Sheets node.<br>4. **APITemplate.io** -- Create an account, design a carousel slide template, and set the template ID in the image generation nodes.<br>5. **Blotato** -- Connect your Instagram, Facebook, and TikTok accounts. Add the Blotato API credential and update account/page IDs in the publishing nodes.<br><br>> **Community node required:** `@blotato/n8n-nodes-blotato` -- this template works on **self-hosted n8n only**.<br>### Step 1: Telegram Input & Voice Transcription<br>User sends a text message or voice note. Voice notes are transcribed via OpenAI Whisper before being passed to the AI agent. |
| Speech to Text | openAi | Transcribes voice note to text | Get Voice File | AI: Draft & Revise Post | ## Create and publish AI carousel posts to Instagram, Facebook, and TikTok<br>Turn a **Telegram message or voice note** into polished carousel posts published simultaneously to **Instagram, Facebook, and TikTok** via Blotato.<br><br>## How it works<br><br>1. Send a topic to the Telegram bot (text or voice note).<br>2. Voice notes are transcribed via OpenAI Whisper.<br>3. An AI Agent (GPT with conversation memory) drafts a script, 5 carousel quotes, and a social media caption with hashtags.<br>4. You review and iterate via Telegram chat until you approve.<br>5. Approved quotes are logged in Google Sheets.<br>6. APITemplate.io generates 5 styled carousel slide images from your template.<br>7. The carousel is published to Instagram, Facebook, and TikTok via Blotato.<br>8. Each platform's publish status is monitored -- you get a Telegram confirmation or error.<br><br>## Setup steps<br><br>1. **Telegram Bot** -- Create via @BotFather, add the API token as credential.<br>2. **OpenAI** -- Add your API key (used for the AI Agent and Whisper transcription).<br>3. **Google Sheets** -- Create OAuth2 credentials. Set up a spreadsheet with columns: run_id, quote1-5, social_media_text. Update the Sheet ID in the Google Sheets node.<br>4. **APITemplate.io** -- Create an account, design a carousel slide template, and set the template ID in the image generation nodes.<br>5. **Blotato** -- Connect your Instagram, Facebook, and TikTok accounts. Add the Blotato API credential and update account/page IDs in the publishing nodes.<br><br>> **Community node required:** `@blotato/n8n-nodes-blotato` -- this template works on **self-hosted n8n only**.<br>### Step 1: Telegram Input & Voice Transcription<br>User sends a text message or voice note. Voice notes are transcribed via OpenAI Whisper before being passed to the AI agent. |
| OpenAI Chat Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | Primary LLM for content generation |  | AI: Draft & Revise Post | ### Step 2: AI Script Writer & Approval<br>An OpenAI-powered AI Agent with conversation memory drafts a script, 5 carousel quotes, and a social media caption. The user iterates via Telegram until approving. Approved quotes are logged in Google Sheets. |
| Window Buffer Memory | @n8n/n8n-nodes-langchain.memoryBufferWindow | Stores recent conversation context per Telegram user |  | AI: Draft & Revise Post | ### Step 2: AI Script Writer & Approval<br>An OpenAI-powered AI Agent with conversation memory drafts a script, 5 carousel quotes, and a social media caption. The user iterates via Telegram until approving. Approved quotes are logged in Google Sheets. |
| Structured Output Parser | @n8n/n8n-nodes-langchain.outputParserStructured | Defines required JSON schema |  | Auto-fixing Output Parser | ### Step 2: AI Script Writer & Approval<br>An OpenAI-powered AI Agent with conversation memory drafts a script, 5 carousel quotes, and a social media caption. The user iterates via Telegram until approving. Approved quotes are logged in Google Sheets. |
| OpenAI Chat Model (parser) | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM used to repair malformed structured output |  | Auto-fixing Output Parser | ### Step 2: AI Script Writer & Approval<br>An OpenAI-powered AI Agent with conversation memory drafts a script, 5 carousel quotes, and a social media caption. The user iterates via Telegram until approving. Approved quotes are logged in Google Sheets. |
| Auto-fixing Output Parser | @n8n/n8n-nodes-langchain.outputParserAutofixing | Repairs malformed JSON against schema | Structured Output Parser; OpenAI Chat Model (parser) | AI: Draft & Revise Post | ### Step 2: AI Script Writer & Approval<br>An OpenAI-powered AI Agent with conversation memory drafts a script, 5 carousel quotes, and a social media caption. The user iterates via Telegram until approving. Approved quotes are logged in Google Sheets. |
| AI: Draft & Revise Post | @n8n/n8n-nodes-langchain.agent | Drafts, revises, and returns approval-state content | Check if input is a voice message; Speech to Text; OpenAI Chat Model; Window Buffer Memory; Auto-fixing Output Parser | Check if Approved | ### Step 2: AI Script Writer & Approval<br>An OpenAI-powered AI Agent with conversation memory drafts a script, 5 carousel quotes, and a social media caption. The user iterates via Telegram until approving. Approved quotes are logged in Google Sheets. |
| Check if Approved | if | Routes approved content to publishing prep or draft back to user | AI: Draft & Revise Post | Extract approved quotes; Send draft to Telegram for review | ### Step 2: AI Script Writer & Approval<br>An OpenAI-powered AI Agent with conversation memory drafts a script, 5 carousel quotes, and a social media caption. The user iterates via Telegram until approving. Approved quotes are logged in Google Sheets. |
| Send draft to Telegram for review | telegram | Sends draft content and follow-up prompt to Telegram user | Check if Approved |  | ### Step 2: AI Script Writer & Approval<br>An OpenAI-powered AI Agent with conversation memory drafts a script, 5 carousel quotes, and a social media caption. The user iterates via Telegram until approving. Approved quotes are logged in Google Sheets. |
| Extract approved quotes | set | Flattens approved quote and caption fields | Check if Approved | Log approved quotes in Google Sheets; Notify user: preparing post | ### Step 3: Image Generation<br>All 5 quotes are sent to APITemplate.io to generate styled carousel slide images using your pre-designed template. Images are then merged and the public urls are collected for the publishing step. |
| Log approved quotes in Google Sheets | googleSheets | Appends approved content log row | Extract approved quotes | Generate carousel slide 1; Generate carousel slide 2; Generate carousel slide 3; Generate carousel slide 4; Generate carousel slide 5 | ### Step 3: Image Generation<br>All 5 quotes are sent to APITemplate.io to generate styled carousel slide images using your pre-designed template. Images are then merged and the public urls are collected for the publishing step. |
| Notify user: preparing post | telegram | Notifies user that media generation is starting | Extract approved quotes |  | ### Step 3: Image Generation<br>All 5 quotes are sent to APITemplate.io to generate styled carousel slide images using your pre-designed template. Images are then merged and the public urls are collected for the publishing step. |
| Generate carousel slide 1 | apiTemplateIo | Creates slide image for quote 1 | Log approved quotes in Google Sheets | Merge all carousel slides | ### Step 3: Image Generation<br>All 5 quotes are sent to APITemplate.io to generate styled carousel slide images using your pre-designed template. Images are then merged and the public urls are collected for the publishing step. |
| Generate carousel slide 2 | apiTemplateIo | Creates slide image for quote 2 | Log approved quotes in Google Sheets | Merge all carousel slides | ### Step 3: Image Generation<br>All 5 quotes are sent to APITemplate.io to generate styled carousel slide images using your pre-designed template. Images are then merged and the public urls are collected for the publishing step. |
| Generate carousel slide 3 | apiTemplateIo | Creates slide image for quote 3 | Log approved quotes in Google Sheets | Merge all carousel slides | ### Step 3: Image Generation<br>All 5 quotes are sent to APITemplate.io to generate styled carousel slide images using your pre-designed template. Images are then merged and the public urls are collected for the publishing step. |
| Generate carousel slide 4 | apiTemplateIo | Creates slide image for quote 4 | Log approved quotes in Google Sheets | Merge all carousel slides | ### Step 3: Image Generation<br>All 5 quotes are sent to APITemplate.io to generate styled carousel slide images using your pre-designed template. Images are then merged and the public urls are collected for the publishing step. |
| Generate carousel slide 5 | apiTemplateIo | Creates slide image for quote 5 | Log approved quotes in Google Sheets | Merge all carousel slides | ### Step 3: Image Generation<br>All 5 quotes are sent to APITemplate.io to generate styled carousel slide images using your pre-designed template. Images are then merged and the public urls are collected for the publishing step. |
| Merge all carousel slides | merge | Waits for all 5 slide generations | Generate carousel slide 1; Generate carousel slide 2; Generate carousel slide 3; Generate carousel slide 4; Generate carousel slide 5 | Collect all image URLs for publishing | ### Step 3: Image Generation<br>All 5 quotes are sent to APITemplate.io to generate styled carousel slide images using your pre-designed template. Images are then merged and the public urls are collected for the publishing step. |
| Collect all image URLs for publishing | set | Aggregates image URLs and caption for publishing | Merge all carousel slides | Create Instagram Post; Create Facebook Post; Create TikTok Post | ### Step 3: Image Generation<br>All 5 quotes are sent to APITemplate.io to generate styled carousel slide images using your pre-designed template. Images are then merged and the public urls are collected for the publishing step. |
| Create Instagram Post | @blotato/n8n-nodes-blotato.blotato | Submits Instagram carousel post | Collect all image URLs for publishing | Wait 25s before checking Instagram | ### Step 4.a: Instagram Publishing<br>Create Instagram post with status monitoring loop that retries while in-progress and sends a Telegram confirmation or error message. |
| Wait 25s before checking Instagram | wait | Initial delay before Instagram status check | Create Instagram Post | Instagram Check Post Status | ### Step 4.a: Instagram Publishing<br>Create Instagram post with status monitoring loop that retries while in-progress and sends a Telegram confirmation or error message. |
| Instagram Check Post Status | @blotato/n8n-nodes-blotato.blotato | Polls Instagram post submission status | Wait 25s before checking Instagram; Retry: wait 5s for Instagram | Instagram post published? | ### Step 4.a: Instagram Publishing<br>Create Instagram post with status monitoring loop that retries while in-progress and sends a Telegram confirmation or error message. |
| Instagram post published? | if | Checks whether Instagram status is published | Instagram Check Post Status | Send Instagram success notification; Instagram still processing? | ### Step 4.a: Instagram Publishing<br>Create Instagram post with status monitoring loop that retries while in-progress and sends a Telegram confirmation or error message. |
| Instagram still processing? | if | Decides retry vs failure for Instagram | Instagram post published? | Retry: wait 5s for Instagram; Send Instagram error notification | ### Step 4.a: Instagram Publishing<br>Create Instagram post with status monitoring loop that retries while in-progress and sends a Telegram confirmation or error message. |
| Retry: wait 5s for Instagram | wait | Delay before Instagram retry poll | Instagram still processing? | Instagram Check Post Status | ### Step 4.a: Instagram Publishing<br>Create Instagram post with status monitoring loop that retries while in-progress and sends a Telegram confirmation or error message. |
| Send Instagram success notification | telegram | Sends Instagram success message with public URL | Instagram post published? |  | ### Step 4.a: Instagram Publishing<br>Create Instagram post with status monitoring loop that retries while in-progress and sends a Telegram confirmation or error message. |
| Send Instagram error notification | telegram | Sends Instagram failure message | Instagram still processing? |  | ### Step 4.a: Instagram Publishing<br>Create Instagram post with status monitoring loop that retries while in-progress and sends a Telegram confirmation or error message. |
| Create Facebook Post | @blotato/n8n-nodes-blotato.blotato | Submits Facebook carousel post | Collect all image URLs for publishing | Wait 10s before checking Facebook | ### Step 4.b: Facebook Publishing<br>Create Facebook post with status monitoring loop that retries while in-progress and sends a Telegram confirmation or error message. |
| Wait 10s before checking Facebook | wait | Initial delay before Facebook status check | Create Facebook Post | Facebook Check Post Status | ### Step 4.b: Facebook Publishing<br>Create Facebook post with status monitoring loop that retries while in-progress and sends a Telegram confirmation or error message. |
| Facebook Check Post Status | @blotato/n8n-nodes-blotato.blotato | Polls Facebook post submission status | Wait 10s before checking Facebook; Retry: wait 5s for Facebook | Facebook post published? | ### Step 4.b: Facebook Publishing<br>Create Facebook post with status monitoring loop that retries while in-progress and sends a Telegram confirmation or error message. |
| Facebook post published? | if | Checks whether Facebook status is published | Facebook Check Post Status | Facebook still processing? | ### Step 4.b: Facebook Publishing<br>Create Facebook post with status monitoring loop that retries while in-progress and sends a Telegram confirmation or error message. |
| Facebook still processing? | if | Decides retry vs failure for Facebook | Facebook post published? | Retry: wait 5s for Facebook; Send Facebook error notification | ### Step 4.b: Facebook Publishing<br>Create Facebook post with status monitoring loop that retries while in-progress and sends a Telegram confirmation or error message. |
| Retry: wait 5s for Facebook | wait | Delay before Facebook retry poll | Facebook still processing? | Facebook Check Post Status | ### Step 4.b: Facebook Publishing<br>Create Facebook post with status monitoring loop that retries while in-progress and sends a Telegram confirmation or error message. |
| Send Facebook error notification | telegram | Sends Facebook failure message | Facebook still processing? |  | ### Step 4.b: Facebook Publishing<br>Create Facebook post with status monitoring loop that retries while in-progress and sends a Telegram confirmation or error message. |
| Create TikTok Post | @blotato/n8n-nodes-blotato.blotato | Submits TikTok carousel post | Collect all image URLs for publishing | Wait 10s before checking TikTok | ### Step 4.c: TikTok Publishing<br>Create TikTok post with status monitoring loop that retries while in-progress and sends a Telegram confirmation or error message. |
| Wait 10s before checking TikTok | wait | Initial delay before TikTok status check | Create TikTok Post | TikTok Check Post Status | ### Step 4.c: TikTok Publishing<br>Create TikTok post with status monitoring loop that retries while in-progress and sends a Telegram confirmation or error message. |
| TikTok Check Post Status | @blotato/n8n-nodes-blotato.blotato | Polls TikTok post submission status | Wait 10s before checking TikTok; Retry: wait 5s for TikTok | TikTok post published? | ### Step 4.c: TikTok Publishing<br>Create TikTok post with status monitoring loop that retries while in-progress and sends a Telegram confirmation or error message. |
| TikTok post published? | if | Checks whether TikTok status is published | TikTok Check Post Status | TikTok still processing? | ### Step 4.c: TikTok Publishing<br>Create TikTok post with status monitoring loop that retries while in-progress and sends a Telegram confirmation or error message. |
| TikTok still processing? | if | Decides retry vs failure for TikTok | TikTok post published? | Retry: wait 5s for TikTok; Send TikTok error notification | ### Step 4.c: TikTok Publishing<br>Create TikTok post with status monitoring loop that retries while in-progress and sends a Telegram confirmation or error message. |
| Retry: wait 5s for TikTok | wait | Delay before TikTok retry poll | TikTok still processing? | TikTok Check Post Status | ### Step 4.c: TikTok Publishing<br>Create TikTok post with status monitoring loop that retries while in-progress and sends a Telegram confirmation or error message. |
| Send TikTok error notification | telegram | Sends TikTok failure message | TikTok still processing? |  | ### Step 4.c: TikTok Publishing<br>Create TikTok post with status monitoring loop that retries while in-progress and sends a Telegram confirmation or error message. |
| Sticky Note | stickyNote | Visual documentation and setup note |  |  |  |
| Sticky Note1 | stickyNote | Visual label for input/transcription block |  |  |  |
| Sticky Note2 | stickyNote | Visual label for AI/approval block |  |  |  |
| Sticky Note3 | stickyNote | Visual label for image generation block |  |  |  |
| Sticky Note4 | stickyNote | Visual label for Instagram block |  |  |  |
| Sticky Note5 | stickyNote | Visual label for Facebook block |  |  |  |
| Sticky Note6 | stickyNote | Visual label for TikTok block |  |  |  |

---

# 4. Reproducing the Workflow from Scratch

Below is a clean rebuild procedure. Where the source workflow contains naming mistakes, the steps below include the **intended working setup**, not the broken references.

## Prerequisites
1. Set up credentials in n8n for:
   - **Telegram Bot API**
   - **OpenAI**
   - **Google Sheets OAuth2**
   - **APITemplate.io**
   - **Blotato**
2. If self-hosted, install the Blotato community node:
   - `@blotato/n8n-nodes-blotato`
3. Create a Google Sheet with columns:
   - `run_id`
   - `quote1`
   - `quote2`
   - `quote3`
   - `quote4`
   - `quote5`
   - `social_media_text`
4. Create one APITemplate.io image template and note the layer key to override, such as `text_1_1_1.text`.
5. Connect your Instagram, Facebook, and TikTok accounts in Blotato and note the needed account/page IDs.

## Build Steps

1. **Create a Telegram Trigger node**
   - Type: **Telegram Trigger**
   - Name: `Start: Telegram Message`
   - Listen to update type: `message`

2. **Create a Set node**
   - Name: `Extract text from Telegram message`
   - Add field:
     - `text` = `{{ $json?.message?.text || "" }}`

3. **Connect** `Start: Telegram Message` → `Extract text from Telegram message`

4. **Create an If node**
   - Name: `Check if input is a voice message`
   - Condition: `{{ $json.message.text }}` is empty

5. **Connect** `Extract text from Telegram message` → `Check if input is a voice message`

6. **Create a Telegram node for file retrieval**
   - Name: `Get Voice File`
   - Resource: `file`
   - File ID: `{{ $('Start: Telegram Message').item.json.message.voice.file_id }}`

7. **Connect** the **true** output of `Check if input is a voice message` → `Get Voice File`

8. **Create an OpenAI transcription node**
   - Name: `Speech to Text`
   - Type: OpenAI
   - Resource: `audio`
   - Operation: `transcribe`

9. **Connect** `Get Voice File` → `Speech to Text`

10. **Create a primary OpenAI chat model node**
    - Name: `OpenAI Chat Model`
    - Model: `gpt-5.1`
    - Response format: `json_object`

11. **Create a Window Buffer Memory node**
    - Name: `Window Buffer Memory`
    - Session ID type: custom key
    - Session key: `{{ $('Start: Telegram Message').first().json.message.from.id }}`
    - Context window length: `10`

12. **Create a Structured Output Parser node**
    - Name: `Structured Output Parser`
    - Use manual schema with:
      - `approved` boolean
      - `quotes` array of strings, max 5
      - `followUpQuestion` string
      - `SocialMediaText` string

13. **Create a parser LLM node**
    - Name: `OpenAI Chat Model (parser)`
    - Model: `gpt-5`
    - Text format: `json_schema`
    - Reuse the same schema as above

14. **Create an Auto-fixing Output Parser node**
    - Name: `Auto-fixing Output Parser`

15. **Connect parser components**
    - `Structured Output Parser` → `Auto-fixing Output Parser` via AI output parser port
    - `OpenAI Chat Model (parser)` → `Auto-fixing Output Parser` via AI language model port

16. **Create the AI Agent node**
    - Name: `AI: Draft & Revise Post`
    - Prompt mode: define manually
    - Input text field: `{{ $json.text }}`
    - Enable output parser
    - Paste the system instructions from the source workflow, including:
      - drafting behavior
      - revision behavior
      - JSON-only approval logic
      - final fields `approved`, `quotes`, `followUpQuestion`, `SocialMediaText`

17. **Connect AI support nodes**
    - `OpenAI Chat Model` → `AI: Draft & Revise Post` via AI language model
    - `Window Buffer Memory` → `AI: Draft & Revise Post` via AI memory
    - `Auto-fixing Output Parser` → `AI: Draft & Revise Post` via AI output parser

18. **Connect content inputs to the AI agent**
    - `Speech to Text` → `AI: Draft & Revise Post`
    - **false** output of `Check if input is a voice message` → `AI: Draft & Revise Post`

19. **Create an If node**
    - Name: `Check if Approved`
    - Condition: `{{ $json.output.approved }}` is true

20. **Connect** `AI: Draft & Revise Post` → `Check if Approved`

21. **Create a Telegram node**
    - Name: `Send draft to Telegram for review`
    - Chat ID: `{{ $('Start: Telegram Message').first().json.message.chat.id }}`
    - Message text: include the five quotes, `SocialMediaText`, and `followUpQuestion`
    - A safer format is to mainly send `followUpQuestion` before approval, since quotes may be empty by design

22. **Connect** false branch of `Check if Approved` → `Send draft to Telegram for review`

23. **Create a Set node**
    - Name: `Extract approved quotes`
    - Create fields:
      - `quote1 = {{ $json.output.quotes[0] }}`
      - `quote2 = {{ $json.output.quotes[1] }}`
      - `quote3 = {{ $json.output.quotes[2] }}`
      - `quote4 = {{ $json.output.quotes[3] }}`
      - `quote5 = {{ $json.output.quotes[4] }}`
      - `social_media_text = {{ $json.output.SocialMediaText }}`

24. **Connect** true branch of `Check if Approved` → `Extract approved quotes`

25. **Create a Telegram node**
    - Name: `Notify user: preparing post`
    - Chat ID: `{{ $('Start: Telegram Message').first().json.message.chat.id }}`
    - Text: `Your post is being prepared.`

26. **Create a Google Sheets node**
    - Name: `Log approved quotes in Google Sheets`
    - Operation: `append`
    - Select your spreadsheet and tab
    - Map:
      - `run_id = {{ $now.toISO() }}`
      - `quote1 = {{ $json.quote1.replace(/^\"|\"$/g, '') }}`
      - same for quote2..quote5
      - `social_media_text = {{ $json.social_media_text.replace(/^\"|\"$/g, '') }}`

27. **Connect**
    - `Extract approved quotes` → `Log approved quotes in Google Sheets`
    - `Extract approved quotes` → `Notify user: preparing post`

28. **Create 5 APITemplate.io nodes**
    - Names:
      - `Generate carousel slide 1`
      - `Generate carousel slide 2`
      - `Generate carousel slide 3`
      - `Generate carousel slide 4`
      - `Generate carousel slide 5`
    - In each:
      - enable download
      - select the same template ID
      - override `text_1_1_1.text` with:
        - slide 1: `{{ $('Extract approved quotes').item.json.quote1 }}`
        - slide 2: `{{ $('Extract approved quotes').item.json.quote2 }}`
        - slide 3: `{{ $('Extract approved quotes').item.json.quote3 }}`
        - slide 4: `{{ $('Extract approved quotes').item.json.quote4 }}`
        - slide 5: `{{ $('Extract approved quotes').item.json.quote5 }}`

29. **Connect** `Log approved quotes in Google Sheets` to all five slide nodes

30. **Create a Merge node**
    - Name: `Merge all carousel slides`
    - Number of inputs: `5`

31. **Connect** each slide node to a separate input on `Merge all carousel slides`

32. **Create a Set node**
    - Name: `Collect all image URLs for publishing`
    - Prefer these working assignments:
      - `img_url_1 = {{ $('Generate carousel slide 1').first().json.download_url }}`
      - `img_url_2 = {{ $('Generate carousel slide 2').first().json.download_url }}`
      - `img_url_3 = {{ $('Generate carousel slide 3').first().json.download_url }}`
      - `img_url_4 = {{ $('Generate carousel slide 4').first().json.download_url }}`
      - `img_url_5 = {{ $('Generate carousel slide 5').first().json.download_url }}`
      - `post_ext = {{ $('Extract approved quotes').item.json.social_media_text }}`
    - Optionally enable retry if you expect eventual consistency

33. **Connect** `Merge all carousel slides` → `Collect all image URLs for publishing`

## Instagram Branch

34. **Create a Blotato node**
    - Name: `Create Instagram Post`
    - Platform: Instagram/default
    - Account ID: your Instagram account
    - Caption text: `{{ $json.post_ext }}`
    - Media URLs: comma-separated `{{ $json.img_url_1 }},{{ $json.img_url_2 }},{{ $json.img_url_3 }},{{ $json.img_url_4 }},{{ $json.img_url_5 }}`

35. **Create a Wait node**
    - Name: `Wait 25s before checking Instagram`
    - Amount: `25 seconds`

36. **Create a Blotato node**
    - Name: `Instagram Check Post Status`
    - Operation: `get`
    - Post submission ID: `{{ $('Create Instagram Post').item.json.postSubmissionId }}`

37. **Create an If node**
    - Name: `Instagram post published?`
    - Condition: `{{ $json.status }}` equals `published`

38. **Create an If node**
    - Name: `Instagram still processing?`
    - Condition: `{{ $json.status }}` equals `in-progress`

39. **Create a Wait node**
    - Name: `Retry: wait 5s for Instagram`
    - Set explicit wait to `5 seconds`

40. **Create Telegram notifications**
    - `Send Instagram success notification`
      - Text: `Post successfully published:\n{{ $json.publicUrl }}`
    - `Send Instagram error notification`
      - Text: `Error while uploading your post to Instagram`

41. **Connect Instagram branch**
    - `Collect all image URLs for publishing` → `Create Instagram Post`
    - `Create Instagram Post` → `Wait 25s before checking Instagram`
    - `Wait 25s before checking Instagram` → `Instagram Check Post Status`
    - `Instagram Check Post Status` → `Instagram post published?`
    - true branch → `Send Instagram success notification`
    - false branch → `Instagram still processing?`
    - true branch → `Retry: wait 5s for Instagram` → `Instagram Check Post Status`
    - false branch → `Send Instagram error notification`

## Facebook Branch

42. **Create a Blotato node**
    - Name: `Create Facebook Post`
    - Platform: `facebook`
    - Account ID: your Facebook-connected account
    - Facebook Page ID: your page
    - Caption and media URL setup same as Instagram

43. **Create a Wait node**
    - Name: `Wait 10s before checking Facebook`
    - Amount: `10 seconds`

44. **Create a Blotato get-status node**
    - Name: `Facebook Check Post Status`
    - Post submission ID: `{{ $('Create Facebook Post').item.json.postSubmissionId }}`

45. **Create two If nodes**
    - `Facebook post published?` → status equals `published`
    - `Facebook still processing?` → status equals `in-progress`

46. **Create retry wait**
    - Name: `Retry: wait 5s for Facebook`
    - Amount: `5 seconds`

47. **Create Telegram notifications**
    - Add at least:
      - `Send Facebook success notification`
      - `Send Facebook error notification`
    - The source workflow lacks success notification and uses wrong error text; fix both in your rebuild

48. **Connect Facebook branch**
    - Similar to Instagram logic
    - Recommended:
      - true published → success notification
      - false → processing check
      - in-progress → retry loop
      - otherwise → error notification

## TikTok Branch

49. **Create a Blotato node**
    - Name: `Create TikTok Post`
    - Platform: `tiktok`
    - Account ID: your TikTok account
    - Caption and media URLs same as above
    - TikTok options:
      - Title: `Follow for more`
      - Auto add music: true
      - Privacy: choose desired value; avoid `SELF_ONLY` if you want public posting
      - Is AI generated: true

50. **Create a Wait node**
    - Name: `Wait 10s before checking TikTok`
    - Amount: `10 seconds`

51. **Create a Blotato get-status node**
    - Name: `TikTok Check Post Status`
    - Post submission ID: `{{ $('Create TikTok Post').item.json.postSubmissionId }}`

52. **Create two If nodes**
    - `TikTok post published?` → status equals `published`
    - `TikTok still processing?` → status equals `in-progress`

53. **Create retry wait**
    - Name: `Retry: wait 5s for TikTok`
    - Amount: `5 seconds`

54. **Create Telegram notifications**
    - `Send TikTok success notification`
    - `Send TikTok error notification`
    - The source workflow only includes error notification and uses incorrect Instagram wording; fix this

55. **Connect TikTok branch**
    - Same pattern as Instagram and Facebook

## Recommended Hardening Changes

56. Add validation after approval:
   - Confirm there are exactly 5 quotes before image generation

57. Add a retry limit counter for each platform status loop:
   - Prevent infinite polling

58. Correct all node name references:
   - Replace `Append row in sheet` with the actual node names you use
   - Replace `Create an image1..5` with the APITemplate node names
   - Ensure `img_url_3` is not accidentally named `=img_url_3`

59. Add success notifications for Facebook and TikTok if desired

60. Activate the workflow only after testing each branch independently.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Community node required: `@blotato/n8n-nodes-blotato` | Works on self-hosted n8n only |
| Telegram Bot should be created via `@BotFather` | Telegram setup |
| Google Sheet expected columns: `run_id`, `quote1`-`quote5`, `social_media_text` | Google Sheets setup |
| APITemplate.io requires a predesigned carousel image template and matching override keys | APITemplate.io setup |
| Blotato account/page IDs must be inserted manually into publishing nodes | Blotato setup |
| The provided workflow contains several broken node-name references that must be corrected before it can run end-to-end | Affects slide generation and publishing aggregation |
| Facebook and TikTok branches do not include success notification nodes in the current workflow | Functional gap vs described behavior |
| Facebook and TikTok error notification texts incorrectly mention Instagram | Message content bug |