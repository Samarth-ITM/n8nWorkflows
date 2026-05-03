Create AI-powered LinkedIn posts from Telegram with GPT-4 and images

https://n8nworkflows.xyz/workflows/create-ai-powered-linkedin-posts-from-telegram-with-gpt-4-and-images-14522


# Create AI-powered LinkedIn posts from Telegram with GPT-4 and images

## 1. Workflow Overview

This workflow turns a Telegram bot into a conversational LinkedIn post assistant. A user sends natural-language messages to the bot, and the workflow classifies the intent, generates or revises LinkedIn post drafts with GPT-4, optionally generates an AI image, asks for approval, and then publishes the final result to LinkedIn.

Typical use cases:
- Draft a LinkedIn post from a topic sent via Telegram
- Revise a draft iteratively through chat feedback
- Generate a professional image for the post
- Approve and publish the post with or without an image
- Reset a conversation or automatically clean up abandoned drafts

The workflow is organized into the following logical blocks:

### 1.1 Telegram Input & Intent Classification
Receives Telegram messages, classifies user intent with an AI agent, and routes execution to the appropriate branch.

### 1.2 Post Generation
Builds a new LinkedIn post draft from the extracted topic, stores it in workflow static data, and sends the draft back to Telegram.

### 1.3 Post Revision
Loads the saved draft, applies user feedback through GPT-4, preserves image state, and returns the revised draft to Telegram.

### 1.4 Image Generation & Approval
Creates an image prompt from the user request, generates an AI image with OpenAI image generation, previews it in Telegram, stores the Telegram file ID, and handles explicit image approval.

### 1.5 Publishing to LinkedIn
When the user approves publication, the workflow checks whether an image is pending or approved, then either publishes text only or publishes the post with an image to LinkedIn.

### 1.6 Reset, Fallback, and Maintenance
Handles unclear requests, unsupported requests, user resets, and scheduled cleanup of stale session data.

---

## 2. Block-by-Block Analysis

## 2.1 Telegram Input & Intent Classification

**Overview:**  
This block is the workflow’s conversational entry point. It receives every Telegram message, classifies the user’s intent using GPT-4 and a structured output parser, then dispatches execution to the proper branch.

**Nodes Involved:**  
- Telegram Trigger
- AI Intent Classifier
- Intent Classification Model
- Intent Output Parser
- Route by Intent

### Node Details

#### Telegram Trigger
- **Type and role:** `n8n-nodes-base.telegramTrigger`; event trigger for incoming Telegram updates.
- **Configuration choices:** Configured to listen to `message` updates only.
- **Key expressions/variables:** Downstream nodes use `$('Telegram Trigger').first().json.message.chat.id` and `$json.message.text`.
- **Input / output connections:** Entry point → AI Intent Classifier.
- **Version-specific requirements:** Type version `1.2`.
- **Potential failures:** Invalid Telegram credentials, webhook registration issues, bot not started by the user, unsupported update payload shape.
- **Sub-workflow reference:** None.

#### AI Intent Classifier
- **Type and role:** `@n8n/n8n-nodes-langchain.agent`; LLM-based intent router.
- **Configuration choices:** Uses a detailed system message defining seven intents:
  - `new_topic`
  - `generate_image`
  - `approve_image`
  - `approve_post`
  - `revise_post`
  - `cancel_reset`
  - `unclear`
  
  It also instructs the model to mark requests to retrieve/edit prior LinkedIn posts as unsupported via `details = retrieve_previous_posts_not_supported`.
- **Key expressions/variables:** Input text is `={{ $json.message.text }}`.
- **Input / output connections:**  
  Input from Telegram Trigger.  
  Uses Intent Classification Model as language model and Intent Output Parser as parser.  
  Main output → Route by Intent.
- **Version-specific requirements:** Type version `3.1`.
- **Potential failures:** LLM output schema mismatch, empty/undefined message text, OpenAI auth/quota issues.
- **Sub-workflow reference:** None.

#### Intent Classification Model
- **Type and role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`; GPT-4 model for intent classification.
- **Configuration choices:** Model is set to `gpt-4`.
- **Key expressions/variables:** None beyond model selection.
- **Input / output connections:** AI language model connection → AI Intent Classifier.
- **Version-specific requirements:** Type version `1.3`.
- **Potential failures:** OpenAI API auth, rate limits, unavailable model.
- **Sub-workflow reference:** None.

#### Intent Output Parser
- **Type and role:** `@n8n/n8n-nodes-langchain.outputParserStructured`; enforces structured JSON output from the classifier.
- **Configuration choices:** Manual JSON schema:
  - `intent` enum
  - `details` string
  - `confidence` enum
- **Key expressions/variables:** Output later accessed as `$json.output.intent` and `$json.output.details`.
- **Input / output connections:** AI parser connection → AI Intent Classifier.
- **Version-specific requirements:** Type version `1.3`.
- **Potential failures:** Model returns malformed output or values outside schema.
- **Sub-workflow reference:** None.

#### Route by Intent
- **Type and role:** `n8n-nodes-base.switch`; routes each message to the right branch.
- **Configuration choices:** Checks `={{ $json.output.intent }}` and creates named outputs:
  - `New Topic`
  - `Generate Image`
  - `Approve Image`
  - `Approve Post`
  - `Revise Post`
  - `Cancel/Reset`
  
  Fallback output is `extra`.
- **Key expressions/variables:** `$json.output.intent`
- **Input / output connections:**  
  Input from AI Intent Classifier.  
  Outputs:
  - New Topic → Prepare Topic from AI Intent
  - Generate Image → Prepare Image Prompt from AI Intent
  - Approve Image → Approve Image
  - Approve Post → Get Saved Draft
  - Revise Post → Prepare Feedback from AI Intent
  - Cancel/Reset → both Handle Unclear Intent and Reset User Session
  - Fallback (`extra`) → effectively unhandled in connections
- **Version-specific requirements:** Type version `3.4`.
- **Potential failures:** Unexpected parsed output, unconnected fallback behavior.
- **Sub-workflow reference:** None.

**Important edge case:**  
The `Cancel/Reset` branch is wired to both `Handle Unclear Intent` and `Reset User Session`. This means a cancel/reset message triggers an unclear/help-style Telegram response as well as a real reset confirmation. That is likely unintended behavior.

---

## 2.2 Post Generation

**Overview:**  
This block converts a classified topic into a LinkedIn-ready draft using GPT-4, stores the draft in workflow static data keyed by Telegram chat ID, and sends the draft to the user for review.

**Nodes Involved:**  
- Prepare Topic from AI Intent
- LinkedIn Post Generator
- OpenAI Chat Model
- Conversation Memory
- Save Draft
- Send a text message

### Node Details

#### Prepare Topic from AI Intent
- **Type and role:** `n8n-nodes-base.code`; extracts the topic from intent-classifier output and normalizes data.
- **Configuration choices:** Reads chat ID from Telegram Trigger and topic from `$json.output.details`.
- **Key expressions/variables:**  
  - `chatId = $('Telegram Trigger').first().json.message.chat.id.toString()`
  - `topic = $json.output.details || ''`
- **Input / output connections:** Input from Route by Intent → LinkedIn Post Generator.
- **Version-specific requirements:** Type version `2`.
- **Potential failures:** Missing `details`, trigger payload shape mismatch.
- **Sub-workflow reference:** None.

#### LinkedIn Post Generator
- **Type and role:** `@n8n/n8n-nodes-langchain.agent`; generates a complete LinkedIn post draft.
- **Configuration choices:**  
  - Prompt text: `={{ $json.topic }}`
  - Uses a long system prompt defining:
    - general-purpose LinkedIn strategist behavior
    - hook/insight/value/CTA structure
    - clear English output
    - hashtags and emoji rules
    - “output only the final post”
- **Key expressions/variables:** Input topic from `$json.topic`.
- **Input / output connections:**  
  Main input from Prepare Topic from AI Intent.  
  AI model input from OpenAI Chat Model.  
  Memory input from Conversation Memory.  
  Main output → Save Draft.
- **Version-specific requirements:** Type version `3.1`.
- **Potential failures:** Model drift from expected format, empty topic, OpenAI limits.
- **Sub-workflow reference:** None.

#### OpenAI Chat Model
- **Type and role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`; GPT-4 model used for post generation and revision.
- **Configuration choices:** Model `gpt-4`.
- **Key expressions/variables:** None.
- **Input / output connections:** AI language model output feeds both LinkedIn Post Generator and Revise LinkedIn Post.
- **Version-specific requirements:** Type version `1.3`.
- **Potential failures:** API auth, cost/quota, deprecated model availability.
- **Sub-workflow reference:** None.

#### Conversation Memory
- **Type and role:** `@n8n/n8n-nodes-langchain.memoryBufferWindow`; short-term per-chat conversational memory.
- **Configuration choices:**  
  - `sessionIdType = customKey`
  - `sessionKey = =={{ $json.chatId }}`
  - `contextWindowLength = 10`
- **Key expressions/variables:** Uses `chatId` as memory partition key.
- **Input / output connections:** AI memory feeds both LinkedIn Post Generator and Revise LinkedIn Post.
- **Version-specific requirements:** Type version `1.3`.
- **Potential failures:** Missing `chatId`, memory not matching intended item context.
- **Sub-workflow reference:** None.

#### Save Draft
- **Type and role:** `n8n-nodes-base.code`; persists the draft in workflow global static data.
- **Configuration choices:** Stores:
  - `postContent`
  - `status: pending`
  - `updatedAt`
  - image-related fields
  
  It intentionally preserves approved image information from a previous session.
- **Key expressions/variables:**  
  - `const data = $getWorkflowStaticData('global')`
  - `chatId = $('Telegram Trigger').first().json.message.chat.id.toString()`
  - `postContent = $json.output`
- **Input / output connections:** Input from LinkedIn Post Generator → Send a text message.
- **Version-specific requirements:** Type version `2`.
- **Potential failures:** Static data growth, malformed LLM output, assumptions about prior data fields.
- **Sub-workflow reference:** None.

#### Send a text message
- **Type and role:** `n8n-nodes-base.telegram`; sends the first draft to the user.
- **Configuration choices:** Markdown-formatted message with instructions:
  - `approve` to post
  - `/image ai..(img description)` to include image
  - free-text feedback for revisions
- **Key expressions/variables:**  
  - `chatId = {{ $json.chatId }}`
  - `{{ $json.postContent }}`
- **Input / output connections:** Input from Save Draft; no downstream connection.
- **Version-specific requirements:** Type version `1.2`.
- **Potential failures:** Markdown formatting issues if generated text contains Telegram-breaking characters, Telegram send permissions.
- **Sub-workflow reference:** None.

---

## 2.3 Post Revision

**Overview:**  
This block handles user feedback on an existing draft. It retrieves the current saved draft, prompts GPT-4 to revise only what the user requested, preserves current image state, and sends the revised draft back to Telegram.

**Nodes Involved:**  
- Prepare Feedback from AI Intent
- Revise LinkedIn Post
- OpenAI Chat Model
- Conversation Memory
- Save Revised Draft
- Send Revised Draft

### Node Details

#### Prepare Feedback from AI Intent
- **Type and role:** `n8n-nodes-base.code`; prepares revision payload from classified feedback.
- **Configuration choices:** Loads current draft from static data using Telegram chat ID.
- **Key expressions/variables:**  
  - `feedback = $json.output.details || ''`
  - `currentDraft = savedDraft.postContent || ''`
- **Input / output connections:** Input from Route by Intent → Revise LinkedIn Post.
- **Version-specific requirements:** Type version `2`.
- **Potential failures:** No existing saved draft, empty feedback, empty `currentDraft`.
- **Sub-workflow reference:** None.

#### Revise LinkedIn Post
- **Type and role:** `@n8n/n8n-nodes-langchain.agent`; revises an existing draft.
- **Configuration choices:** Prompt includes:
  - current draft
  - user feedback
  - instruction to revise accordingly
  
  System prompt explicitly says:
  - treat the post as a draft, not a new post
  - apply only requested changes
  - preserve tone/structure unless asked otherwise
  - output only final revised post
- **Key expressions/variables:** Uses `$json.currentDraft` and `$json.feedback`.
- **Input / output connections:**  
  Main input from Prepare Feedback from AI Intent.  
  AI model from OpenAI Chat Model.  
  AI memory from Conversation Memory.  
  Main output → Save Revised Draft.
- **Version-specific requirements:** Type version `3.1`.
- **Potential failures:** Revising with empty draft, over-rewriting by model, API failures.
- **Sub-workflow reference:** None.

#### Save Revised Draft
- **Type and role:** `n8n-nodes-base.code`; overwrites the saved draft while preserving image information.
- **Configuration choices:** Stores revised `postContent`, resets status to pending, preserves all image fields.
- **Key expressions/variables:**  
  - `chatId = $('Prepare Feedback from AI Intent').first().json.chatId`
  - `postContent = $json.output`
- **Input / output connections:** Input from Revise LinkedIn Post → Send Revised Draft.
- **Version-specific requirements:** Type version `2`.
- **Potential failures:** Missing draft data, malformed agent output.
- **Sub-workflow reference:** None.

#### Send Revised Draft
- **Type and role:** `n8n-nodes-base.telegram`; sends revised draft back to the user.
- **Configuration choices:** Markdown message with same review instructions as the initial draft.
- **Key expressions/variables:** `{{ $json.postContent }}`, `{{ $json.chatId }}`
- **Input / output connections:** Input from Save Revised Draft; no downstream connection.
- **Version-specific requirements:** Type version `1.2`.
- **Potential failures:** Telegram markdown parsing with user/AI content.
- **Sub-workflow reference:** None.

---

## 2.4 Image Generation & Approval

**Overview:**  
This block creates a polished AI image prompt from the user’s image request, generates an image, previews it in Telegram, stores metadata, and requires explicit approval before the image can be included in the LinkedIn post.

**Nodes Involved:**  
- Prepare Image Prompt from AI Intent
- Create Image Prompt
- OpenAI Chat Model1
- Generate AI Image
- Save AI Image
- Send a photo message
- Save Telegram Image File ID
- Get Telegram File
- Approve Image
- Check Image Approval Result
- Confirm Image Approval
- Image Approval Error

### Node Details

#### Prepare Image Prompt from AI Intent
- **Type and role:** `n8n-nodes-base.code`; stores pending image request metadata.
- **Configuration choices:** Saves `imagePrompt`, marks:
  - `imageStatus = pending`
  - `imageApproved = false`
  - `imageSource = ai`
- **Key expressions/variables:**  
  - `chatId = $('Telegram Trigger').first().json.message.chat.id.toString()`
  - `imagePrompt = $json.output.details || ''`
- **Input / output connections:** Input from Route by Intent → Create Image Prompt.
- **Version-specific requirements:** Type version `2`.
- **Potential failures:** Empty image description, existing draft absent, static data assumptions.
- **Sub-workflow reference:** None.

#### Create Image Prompt
- **Type and role:** `@n8n/n8n-nodes-langchain.agent`; transforms a simple image description into a richer generation prompt.
- **Configuration choices:**  
  - Prompt text asks for a detailed prompt for a professional LinkedIn post image.
  - System prompt enforces clean composition, business-friendly style, and no explanation.
- **Key expressions/variables:** `{{ $json.imagePrompt }}`
- **Input / output connections:**  
  Main input from Prepare Image Prompt from AI Intent.  
  AI model input from OpenAI Chat Model1.  
  Main output → Generate AI Image.
- **Version-specific requirements:** Type version `3.1`.
- **Potential failures:** Vague prompt generation, API failures.
- **Sub-workflow reference:** None.

#### OpenAI Chat Model1
- **Type and role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`; GPT-4 used for image prompt generation.
- **Configuration choices:** Model `gpt-4`.
- **Key expressions/variables:** None.
- **Input / output connections:** AI language model output → Create Image Prompt.
- **Version-specific requirements:** Type version `1.3`.
- **Potential failures:** OpenAI auth/rate limits.
- **Sub-workflow reference:** None.

#### Generate AI Image
- **Type and role:** `@n8n/n8n-nodes-langchain.openAi`; OpenAI image generation node.
- **Configuration choices:**  
  - Resource: `image`
  - Prompt: `={{ $json.output }}`
  - Size: `1024x1024`
  - Return image URLs: enabled
- **Key expressions/variables:** Uses the output of Create Image Prompt as the image prompt.
- **Input / output connections:** Input from Create Image Prompt → Save AI Image.
- **Version-specific requirements:** Type version `2.1`.
- **Potential failures:** Image generation quota, safety filtering by provider, prompt rejection, URL retrieval issues.
- **Sub-workflow reference:** None.

#### Save AI Image
- **Type and role:** `n8n-nodes-base.code`; extracts image URL from possible OpenAI response shapes and stores it.
- **Configuration choices:** Checks multiple response paths:
  - `item.json.data?.[0]?.url`
  - `item.json.url`
  - `item.json.output?.[0]?.url`
  - `item.json.image_url`
- **Key expressions/variables:** Uses workflow static data keyed by chat ID.
- **Input / output connections:** Input from Generate AI Image → Send a photo message.
- **Version-specific requirements:** Type version `2`.
- **Potential failures:** No URL found, provider response shape changes.
- **Sub-workflow reference:** None.

#### Send a photo message
- **Type and role:** `n8n-nodes-base.telegram`; sends the generated image preview to the user.
- **Configuration choices:**  
  - Operation: `sendPhoto`
  - File is the image URL
  - Markdown caption asks for `approve image` or regeneration
- **Key expressions/variables:**  
  - `file = {{ $json.imageUrl }}`
  - `chatId = {{ $json.chatId }}`
- **Input / output connections:** Input from Save AI Image → Save Telegram Image File ID.
- **Version-specific requirements:** Type version `1.2`.
- **Potential failures:** Telegram cannot fetch remote URL, invalid URL, caption markdown issues.
- **Sub-workflow reference:** None.

#### Save Telegram Image File ID
- **Type and role:** `n8n-nodes-base.code`; captures Telegram’s stored file ID for the sent photo.
- **Configuration choices:** Reads `result.photo`, takes the largest available photo, and stores its `file_id`.
- **Key expressions/variables:**  
  - `const photos = $input.first().json.result?.photo || []`
  - `const largestPhoto = photos[photos.length - 1]`
- **Input / output connections:** Input from Send a photo message → Get Telegram File.
- **Version-specific requirements:** Type version `2`.
- **Potential failures:** Telegram response missing `result.photo`, no photo sizes returned.
- **Sub-workflow reference:** None.

#### Get Telegram File
- **Type and role:** `n8n-nodes-base.telegram`; fetches file information for the stored Telegram file ID.
- **Configuration choices:** Resource `file`, `fileId = {{ $json.telegramFileId }}`.
- **Key expressions/variables:** Telegram file ID from previous node.
- **Input / output connections:** Input from Save Telegram Image File ID; no downstream main output connected.
- **Version-specific requirements:** Type version `1.2`.
- **Potential failures:** Expired/invalid file ID, Telegram API issues.
- **Sub-workflow reference:** None.

**Important note:**  
This node currently has no effective downstream use. It may be leftover logic or intended for future validation.

#### Approve Image
- **Type and role:** `n8n-nodes-base.code`; marks a pending image as approved.
- **Configuration choices:** Checks whether either `imageUrl` or `telegramFileId` exists. If not, returns an error object. If yes, updates:
  - `imageStatus = approved`
  - `imageApproved = true`
- **Key expressions/variables:** Static data keyed by chat ID.
- **Input / output connections:** Input from Route by Intent → Check Image Approval Result.
- **Version-specific requirements:** Type version `2`.
- **Potential failures:** No pending image exists, corrupted static data.
- **Sub-workflow reference:** None.

#### Check Image Approval Result
- **Type and role:** `n8n-nodes-base.switch`; branches on whether image approval succeeded.
- **Configuration choices:** Checks boolean `={{ $json.ok }}`; output `Success`, fallback `extra`.
- **Key expressions/variables:** `$json.ok`
- **Input / output connections:**  
  Success → Confirm Image Approval  
  Fallback → Image Approval Error
- **Version-specific requirements:** Type version `3.4`.
- **Potential failures:** Non-boolean values, malformed approval result.
- **Sub-workflow reference:** None.

#### Confirm Image Approval
- **Type and role:** `n8n-nodes-base.telegram`; confirms image approval to the user.
- **Configuration choices:** Markdown message telling user they can now send `approve` to publish with image.
- **Key expressions/variables:** `chatId = {{ $json.chatId }}`
- **Input / output connections:** Input from Check Image Approval Result success; no downstream connection.
- **Version-specific requirements:** Type version `1.2`.
- **Potential failures:** Telegram send errors.
- **Sub-workflow reference:** None.

#### Image Approval Error
- **Type and role:** `n8n-nodes-base.telegram`; returns the error message when approval fails.
- **Configuration choices:** Sends `={{ $json.message }}`.
- **Key expressions/variables:** `chatId = {{ $json.chatId }}`
- **Input / output connections:** Input from Check Image Approval Result fallback.
- **Version-specific requirements:** Type version `1.2`.
- **Potential failures:** Telegram send errors.
- **Sub-workflow reference:** None.

---

## 2.5 Publishing to LinkedIn

**Overview:**  
This block is triggered when the user approves the post. It loads the stored draft, blocks publication if an image exists but is not yet approved, otherwise publishes either a text-only LinkedIn post or a post with an image, then clears the session data.

**Nodes Involved:**  
- Get Saved Draft
- Check Final Post Readiness
- Image Not Approved Warning
- Check Image Status
- Post Text Only
- Clear Post Data After Text Post
- Confirmation
- Prepare Post Data
- Download Image for Post
- Add Post Content to Image
- Create a post
- Clear Post Data After Image Post
- Confirmation With Image

### Node Details

#### Get Saved Draft
- **Type and role:** `n8n-nodes-base.code`; retrieves the user’s saved draft and image metadata.
- **Configuration choices:** Returns:
  - `postContent`
  - `status`
  - `imageApproved`
  - `imageStatus`
  - `imageUrl`
  - `telegramFileId`
  - image file metadata placeholders
- **Key expressions/variables:** Uses Telegram chat ID from trigger and workflow static data.
- **Input / output connections:** Input from Route by Intent approve-post branch → Check Final Post Readiness.
- **Version-specific requirements:** Type version `2`.
- **Potential failures:** No draft exists, missing static data, user approves before generating content.
- **Sub-workflow reference:** None.

#### Check Final Post Readiness
- **Type and role:** `n8n-nodes-base.switch`; prevents posting if an image is pending approval.
- **Configuration choices:** If `imageStatus == pending`, routes to `Image Pending`; fallback goes forward.
- **Key expressions/variables:** `={{ $json.imageStatus }}`
- **Input / output connections:**  
  Image Pending → Image Not Approved Warning  
  Fallback → Check Image Status
- **Version-specific requirements:** Type version `3.4`.
- **Potential failures:** Missing `imageStatus`, empty draft but still passes.
- **Sub-workflow reference:** None.

#### Image Not Approved Warning
- **Type and role:** `n8n-nodes-base.telegram`; informs the user that a pending image is not approved.
- **Configuration choices:** Markdown warning instructing to send `approve image`.
- **Key expressions/variables:** `chatId = {{ $json.chatId }}`
- **Input / output connections:** Input from Check Final Post Readiness pending branch.
- **Version-specific requirements:** Type version `1.2`.
- **Potential failures:** Telegram send issues.
- **Sub-workflow reference:** None.

#### Check Image Status
- **Type and role:** `n8n-nodes-base.switch`; decides whether to publish with image or text only.
- **Configuration choices:** If `imageApproved == true`, output `With Image`; fallback to text-only branch.
- **Key expressions/variables:** `={{ $json.imageApproved }}`
- **Input / output connections:**  
  With Image → Prepare Post Data  
  Fallback → Post Text Only
- **Version-specific requirements:** Type version `3.4`.
- **Potential failures:** Boolean type mismatch.
- **Sub-workflow reference:** None.

#### Post Text Only
- **Type and role:** `n8n-nodes-base.linkedIn`; creates a plain text LinkedIn post.
- **Configuration choices:**  
  - Text = `={{ $json.postContent }}`
  - Person = specific LinkedIn member ID `0iutWNMF-o`
- **Key expressions/variables:** `postContent`
- **Input / output connections:** Input from Check Image Status fallback → Clear Post Data After Text Post.
- **Version-specific requirements:** Type version `1`.
- **Potential failures:** OAuth permission issues, invalid member/person ID, LinkedIn API restrictions, empty text.
- **Sub-workflow reference:** None.

#### Clear Post Data After Text Post
- **Type and role:** `n8n-nodes-base.code`; deletes the user’s saved session after successful text-only publication.
- **Configuration choices:** `delete data[chatId]`
- **Key expressions/variables:** `chatId = $('Get Saved Draft').first().json.chatId`
- **Input / output connections:** Input from Post Text Only → Confirmation.
- **Version-specific requirements:** Type version `2`.
- **Potential failures:** None severe; if delete fails silently, stale drafts remain.
- **Sub-workflow reference:** None.

#### Confirmation
- **Type and role:** `n8n-nodes-base.telegram`; confirms text-only posting success.
- **Configuration choices:** Markdown success message.
- **Key expressions/variables:** `chatId = {{ $('Get Saved Draft').first().json.chatId }}`
- **Input / output connections:** Input from Clear Post Data After Text Post.
- **Version-specific requirements:** Type version `1.2`.
- **Potential failures:** Telegram send issues.
- **Sub-workflow reference:** None.

#### Prepare Post Data
- **Type and role:** `n8n-nodes-base.code`; loads the final text and stored Telegram file ID for image posting.
- **Configuration choices:** Returns `chatId`, `postContent`, and `telegramFileId`.
- **Key expressions/variables:** Reads from global static data using chat ID from Get Saved Draft.
- **Input / output connections:** Input from Check Image Status with-image branch → Download Image for Post.
- **Version-specific requirements:** Type version `2`.
- **Potential failures:** Missing `telegramFileId`, empty draft.
- **Sub-workflow reference:** None.

#### Download Image for Post
- **Type and role:** `n8n-nodes-base.telegram`; retrieves the Telegram-hosted file for use in posting.
- **Configuration choices:** Resource `file`, fileId from stored Telegram file ID.
- **Key expressions/variables:** `fileId = {{ $json.telegramFileId }}`
- **Input / output connections:** Input from Prepare Post Data → Add Post Content to Image.
- **Version-specific requirements:** Type version `1.2`.
- **Potential failures:** Invalid Telegram file ID, file inaccessible.
- **Sub-workflow reference:** None.

#### Add Post Content to Image
- **Type and role:** `n8n-nodes-base.set`; preserves binary image data and injects `postContent` into the item.
- **Configuration choices:** Adds a string field:
  - `postContent = {{ $('Prepare Post Data').item.json.postContent }}`
  - Keeps other fields with `includeOtherFields = true`
- **Key expressions/variables:** Pulls text from Prepare Post Data.
- **Input / output connections:** Input from Download Image for Post → Create a post.
- **Version-specific requirements:** Type version `3.4`.
- **Potential failures:** Expression scope mismatch, missing binary file payload.
- **Sub-workflow reference:** None.

#### Create a post
- **Type and role:** `n8n-nodes-base.linkedIn`; publishes a LinkedIn post with image.
- **Configuration choices:**  
  - Text = `={{ $json.postContent }}`
  - Person = `0iutWNMF-o`
  - `shareMediaCategory = IMAGE`
  
  Assumes incoming item includes the required binary/file context from Telegram.
- **Key expressions/variables:** `postContent`
- **Input / output connections:** Input from Add Post Content to Image → Clear Post Data After Image Post.
- **Version-specific requirements:** Type version `1`.
- **Potential failures:** Binary/image payload not in expected shape, LinkedIn OAuth scope issues, unsupported media upload, empty file.
- **Sub-workflow reference:** None.

#### Clear Post Data After Image Post
- **Type and role:** `n8n-nodes-base.code`; deletes saved session after successful image post.
- **Configuration choices:** `delete data[chatId]`
- **Key expressions/variables:** `chatId = $('Get Saved Draft').first().json.chatId`
- **Input / output connections:** Input from Create a post → Confirmation With Image.
- **Version-specific requirements:** Type version `2`.
- **Potential failures:** Stale session if delete fails.
- **Sub-workflow reference:** None.

#### Confirmation With Image
- **Type and role:** `n8n-nodes-base.telegram`; confirms publication with image.
- **Configuration choices:** Markdown success message indicating post was published with image.
- **Key expressions/variables:** `chatId = {{ $('Get Saved Draft').first().json.chatId }}`
- **Input / output connections:** Input from Clear Post Data After Image Post.
- **Version-specific requirements:** Type version `1.2`.
- **Potential failures:** Telegram send issues.
- **Sub-workflow reference:** None.

---

## 2.6 Reset, Fallback, and Maintenance

**Overview:**  
This block handles unsupported or ambiguous messages, allows manual reset of the conversation state, and performs daily cleanup of stale drafts older than 48 hours.

**Nodes Involved:**  
- Handle Unclear Intent
- Reset User Session
- Confirm Reset
- Daily Cleanup Schedule
- Clean Abandoned Drafts

### Node Details

#### Handle Unclear Intent
- **Type and role:** `n8n-nodes-base.telegram`; sends guidance when the intent is unclear or unsupported.
- **Configuration choices:** Conditional expression in the message body:
  - if `details === retrieve_previous_posts_not_supported`, send a specific “Feature Not Available” message
  - otherwise send a generic capability/help message
- **Key expressions/variables:** `$json.output.details`
- **Input / output connections:** Input from Route by Intent cancel/reset branch.
- **Version-specific requirements:** Type version `1.2`.
- **Potential failures:** Because of current routing, this may also send on actual reset requests, which is confusing.
- **Sub-workflow reference:** None.

#### Reset User Session
- **Type and role:** `n8n-nodes-base.code`; wipes all stored data for the current Telegram chat.
- **Configuration choices:** Deletes the static data entry for the chat ID.
- **Key expressions/variables:** `chatId = $('Telegram Trigger').first().json.message.chat.id.toString()`
- **Input / output connections:** Input from Route by Intent cancel/reset branch → Confirm Reset.
- **Version-specific requirements:** Type version `2`.
- **Potential failures:** None severe; if no session exists, delete is harmless.
- **Sub-workflow reference:** None.

#### Confirm Reset
- **Type and role:** `n8n-nodes-base.telegram`; confirms session reset to the user.
- **Configuration choices:** Markdown confirmation telling the user they can start fresh by sending a new topic.
- **Key expressions/variables:** `chatId = {{ $json.chatId }}`
- **Input / output connections:** Input from Reset User Session.
- **Version-specific requirements:** Type version `1.2`.
- **Potential failures:** Telegram send issues.
- **Sub-workflow reference:** None.

#### Daily Cleanup Schedule
- **Type and role:** `n8n-nodes-base.scheduleTrigger`; scheduled maintenance trigger.
- **Configuration choices:** Runs daily at hour `2` (2 AM).
- **Key expressions/variables:** None.
- **Input / output connections:** Trigger → Clean Abandoned Drafts.
- **Version-specific requirements:** Type version `1.2`.
- **Potential failures:** Instance timezone misunderstandings, inactive workflow prevents execution.
- **Sub-workflow reference:** None.

#### Clean Abandoned Drafts
- **Type and role:** `n8n-nodes-base.code`; removes drafts older than 48 hours from static data.
- **Configuration choices:**  
  - `maxAgeHours = 48`
  - compares `updatedAt` to current time
  - returns summary with cleaned count
- **Key expressions/variables:** Uses global static data and `updatedAt`.
- **Input / output connections:** Input from Daily Cleanup Schedule; no downstream node.
- **Version-specific requirements:** Type version `2`.
- **Potential failures:** Missing or malformed `updatedAt` values, static data scale.
- **Sub-workflow reference:** None.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Telegram Trigger | n8n-nodes-base.telegramTrigger | Receives Telegram messages |  | AI Intent Classifier | # 🤖 AI LinkedIn Post Assistant with Telegram<br>## What This Workflow Does<br>Create professional LinkedIn posts through natural conversation with your Telegram bot. The AI generates content, creates images with DALL-E 3, and publishes directly to LinkedIn—all from your phone.<br><br>**How It Works:**<br>1. Message your bot with a topic (e.g., "Write about AI in healthcare")<br>2. Review the AI-generated draft<br>3. Optional: Request an AI image or provide feedback for revisions<br>4. Say "approve" to publish to LinkedIn<br><br>## Setup Instructions<br>☐ **Create Telegram Bot:** Message @BotFather on Telegram, send `/newbot`, and copy your bot token<br>☐ **Add Credentials:** Configure Telegram API, LinkedIn OAuth2, and OpenAI API credentials in n8n<br>☐ **Activate Workflow:** Save and toggle to Active<br>☐ **Test:** Message your bot with a topic to verify it works<br><br>## Customization<br>• **Adjust AI tone:** Edit the "LinkedIn Post Generator" system message for different writing styles<br>• **Change cleanup schedule:** Modify "Daily Cleanup Schedule" (default: 2 AM daily, 48-hour retention)<br>• **Customize responses:** Edit Telegram message nodes to change bot replies<br><br>**Commands:** Just chat naturally say "approve", "approve image", "cancel", or provide feedback like "make it shorter" |
| AI Intent Classifier | @n8n/n8n-nodes-langchain.agent | Classifies incoming Telegram message intent | Telegram Trigger, Intent Classification Model, Intent Output Parser | Route by Intent | ## 🎯 Intent Classification<br>AI analyzes user messages to determine intent (new post, image generation, approval, revision, or cancel) and routes to the appropriate handler. |
| Intent Classification Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | GPT-4 model for intent classification |  | AI Intent Classifier | ## 🎯 Intent Classification<br>AI analyzes user messages to determine intent (new post, image generation, approval, revision, or cancel) and routes to the appropriate handler. |
| Intent Output Parser | @n8n/n8n-nodes-langchain.outputParserStructured | Enforces structured classifier output |  | AI Intent Classifier | ## 🎯 Intent Classification<br>AI analyzes user messages to determine intent (new post, image generation, approval, revision, or cancel) and routes to the appropriate handler. |
| Route by Intent | n8n-nodes-base.switch | Routes execution by detected intent | AI Intent Classifier | Prepare Topic from AI Intent; Prepare Image Prompt from AI Intent; Approve Image; Get Saved Draft; Prepare Feedback from AI Intent; Handle Unclear Intent; Reset User Session | ## 🎯 Intent Classification<br>AI analyzes user messages to determine intent (new post, image generation, approval, revision, or cancel) and routes to the appropriate handler. |
| Prepare Topic from AI Intent | n8n-nodes-base.code | Extracts topic for first draft generation | Route by Intent | LinkedIn Post Generator | ## ✍️ Post Generation & Revision<br><br>1st Draft: GPT-4 creates LinkedIn posts based on user topics, saves drafts to workflow memory, and sends previews via Telegram.<br><br>Revision: Users provide feedback to refine posts. AI revises content while preserving approved images and conversation context. |
| LinkedIn Post Generator | @n8n/n8n-nodes-langchain.agent | Generates LinkedIn post draft | Prepare Topic from AI Intent, OpenAI Chat Model, Conversation Memory | Save Draft | ## ✍️ Post Generation & Revision<br><br>1st Draft: GPT-4 creates LinkedIn posts based on user topics, saves drafts to workflow memory, and sends previews via Telegram.<br><br>Revision: Users provide feedback to refine posts. AI revises content while preserving approved images and conversation context. |
| OpenAI Chat Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | GPT-4 model for post generation/revision |  | LinkedIn Post Generator; Revise LinkedIn Post | ## ✍️ Post Generation & Revision<br><br>1st Draft: GPT-4 creates LinkedIn posts based on user topics, saves drafts to workflow memory, and sends previews via Telegram.<br><br>Revision: Users provide feedback to refine posts. AI revises content while preserving approved images and conversation context. |
| Conversation Memory | @n8n/n8n-nodes-langchain.memoryBufferWindow | Stores recent per-chat context |  | LinkedIn Post Generator; Revise LinkedIn Post | ## ✍️ Post Generation & Revision<br><br>1st Draft: GPT-4 creates LinkedIn posts based on user topics, saves drafts to workflow memory, and sends previews via Telegram.<br><br>Revision: Users provide feedback to refine posts. AI revises content while preserving approved images and conversation context. |
| Save Draft | n8n-nodes-base.code | Saves new draft to static workflow data | LinkedIn Post Generator | Send a text message | ## ✍️ Post Generation & Revision<br><br>1st Draft: GPT-4 creates LinkedIn posts based on user topics, saves drafts to workflow memory, and sends previews via Telegram.<br><br>Revision: Users provide feedback to refine posts. AI revises content while preserving approved images and conversation context. |
| Send a text message | n8n-nodes-base.telegram | Sends initial draft to Telegram | Save Draft |  | ## ✍️ Post Generation & Revision<br><br>1st Draft: GPT-4 creates LinkedIn posts based on user topics, saves drafts to workflow memory, and sends previews via Telegram.<br><br>Revision: Users provide feedback to refine posts. AI revises content while preserving approved images and conversation context. |
| Prepare Feedback from AI Intent | n8n-nodes-base.code | Prepares revision request from user feedback | Route by Intent | Revise LinkedIn Post | ## ✍️ Post Generation & Revision<br><br>1st Draft: GPT-4 creates LinkedIn posts based on user topics, saves drafts to workflow memory, and sends previews via Telegram.<br><br>Revision: Users provide feedback to refine posts. AI revises content while preserving approved images and conversation context. |
| Revise LinkedIn Post | @n8n/n8n-nodes-langchain.agent | Revises existing draft | Prepare Feedback from AI Intent, OpenAI Chat Model, Conversation Memory | Save Revised Draft | ## ✍️ Post Generation & Revision<br><br>1st Draft: GPT-4 creates LinkedIn posts based on user topics, saves drafts to workflow memory, and sends previews via Telegram.<br><br>Revision: Users provide feedback to refine posts. AI revises content while preserving approved images and conversation context. |
| Save Revised Draft | n8n-nodes-base.code | Saves revised draft while preserving image state | Revise LinkedIn Post | Send Revised Draft | ## ✍️ Post Generation & Revision<br><br>1st Draft: GPT-4 creates LinkedIn posts based on user topics, saves drafts to workflow memory, and sends previews via Telegram.<br><br>Revision: Users provide feedback to refine posts. AI revises content while preserving approved images and conversation context. |
| Send Revised Draft | n8n-nodes-base.telegram | Sends revised draft to Telegram | Save Revised Draft |  | ## ✍️ Post Generation & Revision<br><br>1st Draft: GPT-4 creates LinkedIn posts based on user topics, saves drafts to workflow memory, and sends previews via Telegram.<br><br>Revision: Users provide feedback to refine posts. AI revises content while preserving approved images and conversation context. |
| Prepare Image Prompt from AI Intent | n8n-nodes-base.code | Stores raw image request and marks image pending | Route by Intent | Create Image Prompt | ## 🖼️ Image Generation & Approval<br>DALL-E 3 creates professional images based on user descriptions. Images are previewed in Telegram and require approval before posting. |
| Create Image Prompt | @n8n/n8n-nodes-langchain.agent | Converts user image request into detailed AI image prompt | Prepare Image Prompt from AI Intent, OpenAI Chat Model1 | Generate AI Image | ## 🖼️ Image Generation & Approval<br>DALL-E 3 creates professional images based on user descriptions. Images are previewed in Telegram and require approval before posting. |
| OpenAI Chat Model1 | @n8n/n8n-nodes-langchain.lmChatOpenAi | GPT-4 model for image prompt writing |  | Create Image Prompt | ## 🖼️ Image Generation & Approval<br>DALL-E 3 creates professional images based on user descriptions. Images are previewed in Telegram and require approval before posting. |
| Generate AI Image | @n8n/n8n-nodes-langchain.openAi | Generates AI image from prompt | Create Image Prompt | Save AI Image | ## 🖼️ Image Generation & Approval<br>DALL-E 3 creates professional images based on user descriptions. Images are previewed in Telegram and require approval before posting. |
| Save AI Image | n8n-nodes-base.code | Extracts image URL and stores pending image state | Generate AI Image | Send a photo message | ## 🖼️ Image Generation & Approval<br>DALL-E 3 creates professional images based on user descriptions. Images are previewed in Telegram and require approval before posting. |
| Send a photo message | n8n-nodes-base.telegram | Sends generated image preview to Telegram | Save AI Image | Save Telegram Image File ID | ## 🖼️ Image Generation & Approval<br>DALL-E 3 creates professional images based on user descriptions. Images are previewed in Telegram and require approval before posting. |
| Save Telegram Image File ID | n8n-nodes-base.code | Stores Telegram-hosted image file ID | Send a photo message | Get Telegram File | ## 🖼️ Image Generation & Approval<br>DALL-E 3 creates professional images based on user descriptions. Images are previewed in Telegram and require approval before posting. |
| Get Telegram File | n8n-nodes-base.telegram | Fetches Telegram file metadata for generated image | Save Telegram Image File ID |  | ## 🖼️ Image Generation & Approval<br>DALL-E 3 creates professional images based on user descriptions. Images are previewed in Telegram and require approval before posting. |
| Approve Image | n8n-nodes-base.code | Marks image as approved if one exists | Route by Intent | Check Image Approval Result | ## 🖼️ Image Generation & Approval<br>DALL-E 3 creates professional images based on user descriptions. Images are previewed in Telegram and require approval before posting. |
| Check Image Approval Result | n8n-nodes-base.switch | Branches on image approval success/failure | Approve Image | Confirm Image Approval; Image Approval Error | ## 🖼️ Image Generation & Approval<br>DALL-E 3 creates professional images based on user descriptions. Images are previewed in Telegram and require approval before posting. |
| Confirm Image Approval | n8n-nodes-base.telegram | Confirms image approval to user | Check Image Approval Result |  | ## 🖼️ Image Generation & Approval<br>DALL-E 3 creates professional images based on user descriptions. Images are previewed in Telegram and require approval before posting. |
| Image Approval Error | n8n-nodes-base.telegram | Sends image approval error message | Check Image Approval Result |  | ## 🖼️ Image Generation & Approval<br>DALL-E 3 creates professional images based on user descriptions. Images are previewed in Telegram and require approval before posting. |
| Get Saved Draft | n8n-nodes-base.code | Loads saved draft before publishing | Route by Intent | Check Final Post Readiness | ## 📤 Publishing to LinkedIn<br>When user approves, posts are published to LinkedIn with or without images. Session data is cleared after successful posting. |
| Check Final Post Readiness | n8n-nodes-base.switch | Blocks publishing if image is still pending approval | Get Saved Draft | Image Not Approved Warning; Check Image Status | ## 📤 Publishing to LinkedIn<br>When user approves, posts are published to LinkedIn with or without images. Session data is cleared after successful posting. |
| Image Not Approved Warning | n8n-nodes-base.telegram | Warns user to approve pending image first | Check Final Post Readiness |  | ## 📤 Publishing to LinkedIn<br>When user approves, posts are published to LinkedIn with or without images. Session data is cleared after successful posting. |
| Check Image Status | n8n-nodes-base.switch | Chooses text-only vs image publishing | Check Final Post Readiness | Prepare Post Data; Post Text Only | ## 📤 Publishing to LinkedIn<br>When user approves, posts are published to LinkedIn with or without images. Session data is cleared after successful posting. |
| Post Text Only | n8n-nodes-base.linkedIn | Publishes text-only LinkedIn post | Check Image Status | Clear Post Data After Text Post | ## 📤 Publishing to LinkedIn<br>When user approves, posts are published to LinkedIn with or without images. Session data is cleared after successful posting. |
| Clear Post Data After Text Post | n8n-nodes-base.code | Deletes session after text post success | Post Text Only | Confirmation | ## 📤 Publishing to LinkedIn<br>When user approves, posts are published to LinkedIn with or without images. Session data is cleared after successful posting. |
| Confirmation | n8n-nodes-base.telegram | Confirms text-only post success | Clear Post Data After Text Post |  | ## 📤 Publishing to LinkedIn<br>When user approves, posts are published to LinkedIn with or without images. Session data is cleared after successful posting. |
| Prepare Post Data | n8n-nodes-base.code | Loads text and image file ID for image posting | Check Image Status | Download Image for Post | ## 📤 Publishing to LinkedIn<br>When user approves, posts are published to LinkedIn with or without images. Session data is cleared after successful posting. |
| Download Image for Post | n8n-nodes-base.telegram | Fetches Telegram image for LinkedIn upload path | Prepare Post Data | Add Post Content to Image | ## 📤 Publishing to LinkedIn<br>When user approves, posts are published to LinkedIn with or without images. Session data is cleared after successful posting. |
| Add Post Content to Image | n8n-nodes-base.set | Adds post text back onto image item | Download Image for Post | Create a post | ## 📤 Publishing to LinkedIn<br>When user approves, posts are published to LinkedIn with or without images. Session data is cleared after successful posting. |
| Create a post | n8n-nodes-base.linkedIn | Publishes LinkedIn post with image | Add Post Content to Image | Clear Post Data After Image Post | ## 📤 Publishing to LinkedIn<br>When user approves, posts are published to LinkedIn with or without images. Session data is cleared after successful posting. |
| Clear Post Data After Image Post | n8n-nodes-base.code | Deletes session after image post success | Create a post | Confirmation With Image | ## 📤 Publishing to LinkedIn<br>When user approves, posts are published to LinkedIn with or without images. Session data is cleared after successful posting. |
| Confirmation With Image | n8n-nodes-base.telegram | Confirms image post success | Clear Post Data After Image Post |  | ## 📤 Publishing to LinkedIn<br>When user approves, posts are published to LinkedIn with or without images. Session data is cleared after successful posting. |
| Handle Unclear Intent | n8n-nodes-base.telegram | Sends unsupported/unclear request guidance | Route by Intent |  | ## Fallback & Reset the Seession |
| Reset User Session | n8n-nodes-base.code | Clears user session on cancel/reset | Route by Intent | Confirm Reset | ## Fallback & Reset the Seession |
| Confirm Reset | n8n-nodes-base.telegram | Confirms session reset | Reset User Session |  | ## Fallback & Reset the Seession |
| Daily Cleanup Schedule | n8n-nodes-base.scheduleTrigger | Triggers automated cleanup daily |  | Clean Abandoned Drafts | ## 🧹 Automatic Cleanup<br>Daily schedule removes abandoned drafts older than 48 hours to prevent memory bloat. Users can also manually reset with "cancel" command. |
| Clean Abandoned Drafts | n8n-nodes-base.code | Deletes stale drafts older than 48 hours | Daily Cleanup Schedule |  | ## 🧹 Automatic Cleanup<br>Daily schedule removes abandoned drafts older than 48 hours to prevent memory bloat. Users can also manually reset with "cancel" command. |
| Main Overview | n8n-nodes-base.stickyNote | Documentation note |  |  |  |
| Section: Post Generation | n8n-nodes-base.stickyNote | Documentation note |  |  |  |
| Section: Image Generation | n8n-nodes-base.stickyNote | Documentation note |  |  |  |
| Section: Publishing | n8n-nodes-base.stickyNote | Documentation note |  |  |  |
| Section: Maintenance | n8n-nodes-base.stickyNote | Documentation note |  |  |  |
| Section: Intent Classification | n8n-nodes-base.stickyNote | Documentation note |  |  |  |
| Section: Maintenance1 | n8n-nodes-base.stickyNote | Documentation note |  |  |  |

---

## 4. Reproducing the Workflow from Scratch

1. **Create the Telegram trigger**
   - Add a **Telegram Trigger** node.
   - Set it to listen for `message` updates.
   - Connect Telegram credentials from your bot token.
   - Save and later activate the workflow so the webhook registers.

2. **Create the AI intent-classification layer**
   - Add an **AI Agent** node named **AI Intent Classifier**.
   - Set input text to the incoming Telegram text:
     - `{{ $json.message.text }}`
   - Paste a system prompt that classifies messages into:
     - `new_topic`
     - `generate_image`
     - `approve_image`
     - `approve_post`
     - `revise_post`
     - `cancel_reset`
     - `unclear`
   - Include instruction to return unsupported requests for previous LinkedIn posts with `details = retrieve_previous_posts_not_supported`.

3. **Attach a GPT-4 model for classification**
   - Add an **OpenAI Chat Model** node.
   - Select model `gpt-4`.
   - Connect it to the **AI Intent Classifier** using the AI language model connection.

4. **Add a structured output parser**
   - Add a **Structured Output Parser** node.
   - Use a manual JSON schema with:
     - `intent` as enum
     - `details` as string
     - `confidence` as enum (`high`, `medium`, `low`)
   - Connect it to the **AI Intent Classifier** as the output parser.

5. **Add an intent router**
   - Add a **Switch** node named **Route by Intent**.
   - Route on `{{ $json.output.intent }}`.
   - Create outputs for:
     - `new_topic`
     - `generate_image`
     - `approve_image`
     - `approve_post`
     - `revise_post`
     - `cancel_reset`
   - Leave a fallback output for unhandled cases if desired.

6. **Connect the entry flow**
   - Connect:
     - **Telegram Trigger** → **AI Intent Classifier**
     - **AI Intent Classifier** → **Route by Intent**

7. **Create the topic-preparation node**
   - Add a **Code** node named **Prepare Topic from AI Intent**.
   - Output:
     - `chatId` from Telegram Trigger chat ID as string
     - `topic` from `$json.output.details`
   - Connect the `new_topic` output of **Route by Intent** to this node.

8. **Create the LinkedIn draft generator**
   - Add an **AI Agent** node named **LinkedIn Post Generator**.
   - Set prompt input to `{{ $json.topic }}`.
   - Add the long-form LinkedIn strategist system prompt defining:
     - target audience
     - hook/insight/value/CTA structure
     - natural English
     - complete ready-to-post output only
   - Connect **Prepare Topic from AI Intent** to this node.

9. **Attach shared GPT-4 model for writing**
   - Reuse or add an **OpenAI Chat Model** node set to `gpt-4`.
   - Connect it to **LinkedIn Post Generator** as AI language model.

10. **Add conversation memory**
    - Add a **Memory Buffer Window** node named **Conversation Memory**.
    - Set:
      - `sessionIdType = customKey`
      - `sessionKey = {{ $json.chatId }}`
      - `contextWindowLength = 10`
    - Connect memory to **LinkedIn Post Generator**.
    - Also reuse this same memory later for revision.

11. **Store generated drafts**
    - Add a **Code** node named **Save Draft**.
    - Use `$getWorkflowStaticData('global')`.
    - Store per `chatId`:
      - `postContent`
      - `status = pending`
      - `updatedAt`
      - image state fields such as `imageStatus`, `imageApproved`, `imageSource`, `imagePrompt`, `telegramFileId`, `imageUrl`, `imageMimeType`, `imageFileName`
    - Preserve approved image info from previous session if present.
    - Connect **LinkedIn Post Generator** → **Save Draft**.

12. **Send initial draft to Telegram**
    - Add a **Telegram** node named **Send a text message**.
    - Send a Markdown message including:
      - the draft content
      - instructions to `approve`
      - instructions to request an image
      - revision feedback instructions
    - Use `chatId = {{ $json.chatId }}`
    - Connect **Save Draft** → **Send a text message**.

13. **Create revision preparation**
    - Add a **Code** node named **Prepare Feedback from AI Intent**.
    - Load static data by `chatId`.
    - Output:
      - `chatId`
      - `feedback = $json.output.details`
      - `currentDraft = savedDraft.postContent`
      - `status`
    - Connect `revise_post` output of **Route by Intent** to this node.

14. **Create the revision AI agent**
    - Add an **AI Agent** node named **Revise LinkedIn Post**.
    - Prompt should include current draft and user feedback.
    - System prompt should instruct the model to:
      - revise only what was requested
      - preserve tone and structure
      - output only final revised post
    - Connect:
      - **Prepare Feedback from AI Intent** → **Revise LinkedIn Post**
      - shared **OpenAI Chat Model** → **Revise LinkedIn Post**
      - **Conversation Memory** → **Revise LinkedIn Post**

15. **Save revised draft**
    - Add a **Code** node named **Save Revised Draft**.
    - Overwrite `postContent` in static data.
    - Preserve existing image state fields.
    - Connect **Revise LinkedIn Post** → **Save Revised Draft**.

16. **Send revised draft**
    - Add a **Telegram** node named **Send Revised Draft**.
    - Use a Markdown message similar to the first draft message.
    - Connect **Save Revised Draft** → **Send Revised Draft**.

17. **Create image-request preparation**
    - Add a **Code** node named **Prepare Image Prompt from AI Intent**.
    - Read chat ID and `$json.output.details`.
    - Store in static data:
      - `imagePrompt`
      - `imageStatus = pending`
      - `imageApproved = false`
      - `imageSource = ai`
      - `updatedAt`
    - Connect `generate_image` output of **Route by Intent** to this node.

18. **Create an AI image-prompt writer**
    - Add an **AI Agent** node named **Create Image Prompt**.
    - Prompt: create a detailed professional LinkedIn image prompt based on the user request.
    - System prompt should enforce polished, business-friendly images and output only the final prompt.
    - Connect **Prepare Image Prompt from AI Intent** → **Create Image Prompt**.

19. **Attach GPT-4 to image-prompt generation**
    - Add or reuse another **OpenAI Chat Model** node named **OpenAI Chat Model1** set to `gpt-4`.
    - Connect it to **Create Image Prompt**.

20. **Generate image**
    - Add an **OpenAI** image-generation node named **Generate AI Image**.
    - Set:
      - Resource = `image`
      - Prompt = `{{ $json.output }}`
      - Size = `1024x1024`
      - Return image URLs = true
    - Connect **Create Image Prompt** → **Generate AI Image**.

21. **Save generated image metadata**
    - Add a **Code** node named **Save AI Image**.
    - Read the image URL from possible response paths.
    - Save in static data:
      - `imageStatus = pending`
      - `imageApproved = false`
      - `imageSource = ai`
      - `imageUrl`
      - `updatedAt`
    - Connect **Generate AI Image** → **Save AI Image**.

22. **Preview image in Telegram**
    - Add a **Telegram** node named **Send a photo message**.
    - Set operation to `sendPhoto`.
    - `file = {{ $json.imageUrl }}`
    - Add a Markdown caption telling the user to send `approve image` or regenerate.
    - Connect **Save AI Image** → **Send a photo message**.

23. **Capture Telegram file ID**
    - Add a **Code** node named **Save Telegram Image File ID**.
    - Read `result.photo`, take the largest photo, and store its `file_id` into static data as `telegramFileId`.
    - Connect **Send a photo message** → **Save Telegram Image File ID**.

24. **Optionally fetch Telegram file metadata**
    - Add a **Telegram** node named **Get Telegram File**.
    - Resource = `file`
    - `fileId = {{ $json.telegramFileId }}`
    - Connect **Save Telegram Image File ID** → **Get Telegram File**
    - This node is not required for the core logic, but exists in the original workflow.

25. **Create image approval logic**
    - Add a **Code** node named **Approve Image**.
    - Load the current user session from static data.
    - If neither `imageUrl` nor `telegramFileId` exists, return:
      - `ok = false`
      - user-facing error message
    - Else set:
      - `imageStatus = approved`
      - `imageApproved = true`
      - `updatedAt`
      - `ok = true`
    - Connect `approve_image` output of **Route by Intent** → **Approve Image**.

26. **Branch on approval success**
    - Add a **Switch** node named **Check Image Approval Result**.
    - Route on boolean `{{ $json.ok }}`
    - `true` branch → success
    - fallback → error
    - Connect **Approve Image** → **Check Image Approval Result**.

27. **Send approval success and failure messages**
    - Add a **Telegram** node named **Confirm Image Approval** for success.
    - Add a **Telegram** node named **Image Approval Error** for failure.
    - Connect both outputs from **Check Image Approval Result**.

28. **Prepare publish flow**
    - Add a **Code** node named **Get Saved Draft**.
    - Load draft and image metadata from static data by chat ID.
    - Output:
      - `chatId`
      - `postContent`
      - `status`
      - `imageApproved`
      - `imageStatus`
      - `imageUrl`
      - `telegramFileId`
      - optional mime/name fields
    - Connect `approve_post` output of **Route by Intent** → **Get Saved Draft**.

29. **Block publication if image is still pending**
    - Add a **Switch** node named **Check Final Post Readiness**.
    - If `{{ $json.imageStatus }}` equals `pending`, go to a warning message.
    - Fallback goes forward.
    - Connect **Get Saved Draft** → **Check Final Post Readiness**.

30. **Add pending-image warning**
    - Add a **Telegram** node named **Image Not Approved Warning**.
    - Message should instruct user to send `approve image`.
    - Connect pending branch of **Check Final Post Readiness** to it.

31. **Choose text-only vs image publication**
    - Add a **Switch** node named **Check Image Status**.
    - If `{{ $json.imageApproved }}` is true, go to image posting.
    - Fallback goes to text-only posting.
    - Connect fallback output of **Check Final Post Readiness** → **Check Image Status**.

32. **Create text-only LinkedIn post node**
    - Add a **LinkedIn** node named **Post Text Only**.
    - Set text to `{{ $json.postContent }}`
    - Set the LinkedIn person/member ID to your profile ID.
    - Connect text-only branch of **Check Image Status** → **Post Text Only**.

33. **Clear session after text-only post**
    - Add a **Code** node named **Clear Post Data After Text Post**.
    - Delete the user’s static data entry by `chatId`.
    - Connect **Post Text Only** → **Clear Post Data After Text Post**.

34. **Send text-only success confirmation**
    - Add a **Telegram** node named **Confirmation**.
    - Send a success message to the original chat.
    - Connect **Clear Post Data After Text Post** → **Confirmation**.

35. **Prepare image post data**
    - Add a **Code** node named **Prepare Post Data**.
    - Read final saved draft from static data.
    - Output:
      - `chatId`
      - `postContent`
      - `telegramFileId`
    - Connect image branch of **Check Image Status** → **Prepare Post Data**.

36. **Download image from Telegram**
    - Add a **Telegram** node named **Download Image for Post**.
    - Resource = `file`
    - `fileId = {{ $json.telegramFileId }}`
    - Connect **Prepare Post Data** → **Download Image for Post**.

37. **Add post text onto the downloaded image item**
    - Add a **Set** node named **Add Post Content to Image**.
    - Keep other fields.
    - Add `postContent = {{ $('Prepare Post Data').item.json.postContent }}`
    - Connect **Download Image for Post** → **Add Post Content to Image**.

38. **Create LinkedIn post with image**
    - Add a **LinkedIn** node named **Create a post**.
    - Set:
      - text = `{{ $json.postContent }}`
      - person/member ID = your LinkedIn user ID
      - share media category = `IMAGE`
    - Ensure the incoming item includes the image file/binary in the shape expected by your n8n LinkedIn node version.
    - Connect **Add Post Content to Image** → **Create a post**.

39. **Clear session after image post**
    - Add a **Code** node named **Clear Post Data After Image Post**.
    - Delete the user’s static data entry.
    - Connect **Create a post** → **Clear Post Data After Image Post**.

40. **Send image-post success confirmation**
    - Add a **Telegram** node named **Confirmation With Image**.
    - Connect **Clear Post Data After Image Post** → **Confirmation With Image**.

41. **Add unclear/unsupported response**
    - Add a **Telegram** node named **Handle Unclear Intent**.
    - Use a conditional expression:
      - if `details = retrieve_previous_posts_not_supported`, explain that only new post creation is supported
      - otherwise show available commands and examples
    - In a corrected implementation, connect the `unclear` fallback branch to this node.

42. **Add manual reset**
    - Add a **Code** node named **Reset User Session**.
    - Delete the current `chatId` entry from static data.
    - Connect `cancel_reset` output of **Route by Intent** → **Reset User Session**.

43. **Confirm reset**
    - Add a **Telegram** node named **Confirm Reset**.
    - Tell the user all drafts/images were cleared and they can start fresh.
    - Connect **Reset User Session** → **Confirm Reset**.

44. **Add daily cleanup trigger**
    - Add a **Schedule Trigger** node named **Daily Cleanup Schedule**.
    - Configure it to run daily at 2 AM.

45. **Add cleanup code**
    - Add a **Code** node named **Clean Abandoned Drafts**.
    - Load workflow static data.
    - Delete entries where `updatedAt` is older than 48 hours.
    - Return summary fields like:
      - `cleanedCount`
      - `timestamp`
      - `message`
    - Connect **Daily Cleanup Schedule** → **Clean Abandoned Drafts**.

46. **Configure credentials**
    - **Telegram API**
      - Use your bot token from BotFather.
      - Ensure the bot has been started by the target Telegram user.
    - **OpenAI API**
      - Provide an API key with access to GPT-4 and image generation.
    - **LinkedIn OAuth2**
      - Authenticate the LinkedIn node with posting permissions.
      - Confirm the correct person/member ID is used in both LinkedIn nodes.

47. **Activate and test**
    - Save workflow.
    - Activate it.
    - Test sequence:
      1. Send a topic to the Telegram bot
      2. Receive draft
      3. Request revision
      4. Generate image
      5. Approve image
      6. Approve final post
      7. Confirm publication on LinkedIn

### Recommended corrections when rebuilding
- Route actual `unclear` intent to **Handle Unclear Intent**.
- Route only `cancel_reset` to **Reset User Session**.
- Optionally add explicit validation if user approves a post before any draft exists.
- Verify the LinkedIn image-post node receives binary data in the correct format for your n8n version.

### Sub-workflow setup
This workflow does **not** invoke any sub-workflows. There are no Execute Workflow nodes.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Create professional LinkedIn posts through natural conversation with your Telegram bot. The AI generates content, creates images with DALL-E 3, and publishes directly to LinkedIn—all from your phone. | Main workflow purpose |
| Setup reminder: create a Telegram bot via `@BotFather`, configure Telegram API, LinkedIn OAuth2, and OpenAI API credentials in n8n, then activate the workflow. | Operational setup |
| Customization note: edit the **LinkedIn Post Generator** system message to change voice, tone, or structure. | Content customization |
| Cleanup defaults: scheduled daily at 2 AM, drafts retained for 48 hours before removal. | Maintenance behavior |
| Supported natural commands include `approve`, `approve image`, `cancel`, and free-text revision requests such as “make it shorter”. | User interaction model |
| Sticky note title contains a typo: `Fallback & Reset the Seession`. | Documentation quality note |
| The workflow uses workflow global static data to store per-chat drafts and image state. This makes it simple, but state persistence depends on n8n instance behavior and is not ideal for high-scale or multi-instance deployments. | Architecture note |
| The image flow stores Telegram `file_id` after previewing the AI-generated image, then later reuses Telegram as the source for LinkedIn image posting. | Image publishing design |
| Current routing appears to send both an unclear-intent help message and a reset action on `cancel_reset`. This should likely be separated during refinement. | Important implementation note |