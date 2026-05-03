Auto-create Instagram carousel posts from Canva with OpenAI and Blotato

https://n8nworkflows.xyz/workflows/auto-create-instagram-carousel-posts-from-canva-with-openai-and-blotato-13904


# Auto-create Instagram carousel posts from Canva with OpenAI and Blotato

# 1. Workflow Overview

This workflow automatically converts Canva-exported carousel slide images into an Instagram carousel post. It receives a webhook payload containing slide data, uploads each image to Blotato, aggregates the uploaded media URLs, generates a caption with OpenAI, requests human approval through Telegram, and finally publishes the post via Blotato if approved.

Typical use cases include automated social media publishing, AI-assisted content pipelines, agency content operations, and creator workflows where Canva assets are produced upstream and n8n handles the publishing sequence.

## 1.1 Input Reception and Slide Expansion
The workflow starts with an HTTP webhook that accepts a POST payload. It extracts the `body.slides` array and turns each slide into an individual item so they can be processed one by one.

## 1.2 Media Upload and Carousel URL Aggregation
Each slide’s `export_url` is uploaded through Blotato as media. The workflow then aggregates the returned uploaded media URLs into a single list suitable for a carousel post.

## 1.3 AI Caption Generation and Human Approval
Once all media URLs are collected, the workflow asks OpenAI to generate an Instagram caption for Soulfit Pilates. The generated caption and the first five media URLs are then sent to Telegram using a send-and-wait approval flow.

## 1.4 Approval Routing and Post Publication
If the Telegram approval response indicates approval, the workflow creates the Instagram post in Blotato using the caption and aggregated media URLs. If approval is denied, it sends a follow-up Telegram message asking the user to check the Canva files.

---

# 2. Block-by-Block Analysis

## Block 1 — Input Reception and Slide Expansion

### Overview
This block receives the upstream automation payload and normalizes the slide list into individual items. It is the entry point of the workflow and prepares one item per Canva slide for downstream upload.

### Nodes Involved
- Webhook
- Split Out Slides

### Node Details

#### Webhook
- **Type and technical role:** `n8n-nodes-base.webhook`  
  Entry point that exposes an HTTP endpoint.
- **Configuration choices:**
  - HTTP method: `POST`
  - Path: `canva-pngs-to-drive`
  - No additional options configured
- **Key expressions or variables used:**
  - None directly in the node, but downstream nodes expect the request body to contain `body.slides`.
- **Input and output connections:**
  - No input node
  - Output → `Split Out Slides`
- **Version-specific requirements:**
  - Uses `typeVersion: 2`
  - In n8n, webhook test URLs and production URLs differ; activation is required for production usage.
- **Edge cases or potential failure types:**
  - Wrong HTTP method
  - Upstream payload missing `body.slides`
  - Invalid JSON payload
  - Webhook not active when using production URL
  - Mismatch between expected body structure and actual webhook sender format
- **Sub-workflow reference:**
  - None

#### Split Out Slides
- **Type and technical role:** `n8n-nodes-base.splitOut`  
  Splits an array field into one item per element.
- **Configuration choices:**
  - Field to split out: `body.slides`
- **Key expressions or variables used:**
  - Reads `body.slides` from the webhook payload
- **Input and output connections:**
  - Input ← `Webhook`
  - Output → `Upload media`
- **Version-specific requirements:**
  - Uses `typeVersion: 1`
  - Requires the `Split Out` node to be available in the installed n8n version.
- **Edge cases or potential failure types:**
  - `body.slides` missing or not an array
  - Empty slide list resulting in no downstream items
  - Payload shape variation, for example if the sender places slides at `slides` instead of `body.slides`
- **Sub-workflow reference:**
  - None

---

## Block 2 — Media Upload and Carousel URL Aggregation

### Overview
This block uploads each slide image to Blotato and collects the resulting media URLs into one array. That array becomes the media payload for the Instagram carousel post.

### Nodes Involved
- Upload media
- Aggregate

### Node Details

#### Upload media
- **Type and technical role:** `@blotato/n8n-nodes-blotato.blotato`  
  Blotato integration node used here for media upload.
- **Configuration choices:**
  - Resource: `media`
  - Media URL: `{{ $json.export_url }}`
- **Key expressions or variables used:**
  - `{{$json.export_url}}`  
    Expects each split slide item to contain a direct image export URL.
- **Input and output connections:**
  - Input ← `Split Out Slides`
  - Output → `Aggregate`
- **Version-specific requirements:**
  - Uses `typeVersion: 2`
  - Requires the Blotato community node/package to be installed in the n8n instance.
  - Requires valid Blotato API credentials configured in n8n.
- **Edge cases or potential failure types:**
  - Missing `export_url`
  - Invalid or inaccessible Canva image URL
  - Blotato authentication failure
  - Upload timeout or unsupported file format
  - Rate limiting if many slides are uploaded rapidly
- **Sub-workflow reference:**
  - None

#### Aggregate
- **Type and technical role:** `n8n-nodes-base.aggregate`  
  Combines values from multiple items into a single output item.
- **Configuration choices:**
  - Aggregates the field `url`
  - Produces an array of uploaded media URLs
- **Key expressions or variables used:**
  - Downstream nodes use `$('Aggregate').item.json.url`
- **Input and output connections:**
  - Input ← `Upload media`
  - Output → `Caption Generator`
- **Version-specific requirements:**
  - Uses `typeVersion: 1`
- **Edge cases or potential failure types:**
  - Upstream upload node not returning a `url` field
  - Partial upload failures causing an incomplete carousel
  - Empty aggregation if no slides were uploaded
- **Sub-workflow reference:**
  - None

---

## Block 3 — AI Caption Generation and Human Approval

### Overview
This block generates an Instagram caption using OpenAI and submits both caption and image URLs to Telegram for approval. It acts as the quality-control checkpoint before publication.

### Nodes Involved
- Caption Generator
- Send message and wait for response

### Node Details

#### Caption Generator
- **Type and technical role:** `@n8n/n8n-nodes-langchain.openAi`  
  OpenAI chat/model node used to generate a caption.
- **Configuration choices:**
  - Model: `gpt-5-mini`
  - Prompt content includes:
    - `Create an effective caption for Soulfit Pilates. Output only the caption and hashtags without other marks or explanations.`
    - `You're a caption generator agent for Soulfit Pilates. Your job is to come up with Instagram Carousel posts captions to go with the Carousel`
  - No built-in tools enabled
- **Key expressions or variables used:**
  - No dynamic prompt variables are inserted from prior nodes
  - Downstream extraction expects:
    - `$('Caption Generator').item.json.output[0].content[0].text`
- **Input and output connections:**
  - Input ← `Aggregate`
  - Output → `Send message and wait for response`
- **Version-specific requirements:**
  - Uses `typeVersion: 2.1`
  - Requires OpenAI credentials configured in n8n
  - Requires a compatible n8n version supporting the LangChain OpenAI node and selected model list
- **Edge cases or potential failure types:**
  - Invalid OpenAI credentials
  - Model unavailable in the connected account or region
  - Output structure changing and breaking downstream expression paths
  - Prompt may generate content outside branding expectations because slide content is not passed into the prompt
- **Sub-workflow reference:**
  - None

#### Send message and wait for response
- **Type and technical role:** `n8n-nodes-base.telegram`  
  Sends a Telegram message and pauses execution until approval/rejection is received.
- **Configuration choices:**
  - Operation: `sendAndWait`
  - Chat ID: placeholder `REPLACE_WITH_YOUR_ACCOUNT_ID`
  - Message body includes:
    - The first five aggregated image URLs
    - The AI-generated caption
  - Attribution disabled
  - Approval type: `double`
- **Key expressions or variables used:**
  - `{{$('Aggregate').item.json.url[0]}}` through `url[4]`
  - `{{$json.output[0].content[0].text}}`
- **Input and output connections:**
  - Input ← `Caption Generator`
  - Output → `If`
- **Version-specific requirements:**
  - Uses `typeVersion: 1.2`
  - Requires Telegram bot credentials
  - `sendAndWait` requires support for approval interactions in the installed n8n version.
- **Edge cases or potential failure types:**
  - Invalid Telegram bot credentials
  - Wrong chat ID
  - Telegram bot cannot message the target user/channel
  - Aggregated URL array shorter than five items, causing undefined values in the message
  - Approval timeout or no user response
  - Output schema may differ depending on Telegram approval response structure
- **Sub-workflow reference:**
  - None

---

## Block 4 — Approval Routing and Post Publication

### Overview
This block decides whether to publish or reject the post based on the Telegram approval response. Approved posts are sent to Blotato for publication; rejected posts trigger a notification and then end.

### Nodes Involved
- If
- Create post
- Send a text message
- No Operation, do nothing

### Node Details

#### If
- **Type and technical role:** `n8n-nodes-base.if`  
  Conditional router for approval status.
- **Configuration choices:**
  - Checks whether `{{$json.data.approved}}` equals `true`
  - Strict boolean comparison
- **Key expressions or variables used:**
  - `{{$json.data.approved}}`
- **Input and output connections:**
  - Input ← `Send message and wait for response`
  - True output → `Create post`
  - False output → `Send a text message`
- **Version-specific requirements:**
  - Uses `typeVersion: 2.3`
- **Edge cases or potential failure types:**
  - Approval payload structure may not contain `data.approved`
  - Type mismatch if the returned value is a string instead of boolean
  - Any Telegram approval schema change would break routing
- **Sub-workflow reference:**
  - None

#### Create post
- **Type and technical role:** `@blotato/n8n-nodes-blotato.blotato`  
  Blotato integration node used to create the final social media post.
- **Configuration choices:**
  - Account ID: placeholder `REPLACE_WITH_YOUR_ACCOUNT_ID`
  - Post text: `{{ $('Caption Generator').item.json.output[0].content[0].text }}`
  - Post media URLs: `{{ $('Aggregate').item.json.url }}`
  - No extra options configured
- **Key expressions or variables used:**
  - `$('Caption Generator').item.json.output[0].content[0].text`
  - `$('Aggregate').item.json.url`
- **Input and output connections:**
  - Input ← `If` (true branch)
  - No downstream node
- **Version-specific requirements:**
  - Uses `typeVersion: 2`
  - Requires Blotato credentials
  - Requires a valid connected social account in Blotato matching the account ID
- **Edge cases or potential failure types:**
  - Placeholder account ID not replaced
  - Blotato auth failure
  - Media URL format not accepted by Blotato post creation endpoint
  - Caption too long for the target platform
  - Missing or malformed aggregated media array
- **Sub-workflow reference:**
  - None

#### Send a text message
- **Type and technical role:** `n8n-nodes-base.telegram`  
  Sends a rejection/fallback message after a non-approved result.
- **Configuration choices:**
  - Chat ID: placeholder `REPLACE_WITH_YOUR_ACCOUNT_ID`
  - Text: `Got it. In that case, please check your Canva files`
  - Attribution disabled
- **Key expressions or variables used:**
  - None
- **Input and output connections:**
  - Input ← `If` (false branch)
  - Output → `No Operation, do nothing`
- **Version-specific requirements:**
  - Uses `typeVersion: 1.2`
  - Requires Telegram credentials
- **Edge cases or potential failure types:**
  - Wrong chat ID
  - Telegram permission issues
  - Bot blocked by recipient
- **Sub-workflow reference:**
  - None

#### No Operation, do nothing
- **Type and technical role:** `n8n-nodes-base.noOp`  
  Explicit terminal node for the rejection branch.
- **Configuration choices:**
  - No parameters
- **Key expressions or variables used:**
  - None
- **Input and output connections:**
  - Input ← `Send a text message`
  - No output
- **Version-specific requirements:**
  - Uses `typeVersion: 1`
- **Edge cases or potential failure types:**
  - None functionally; this node simply ends the branch.
- **Sub-workflow reference:**
  - None

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Webhook | n8n-nodes-base.webhook | Receives POST payload containing Canva slide data |  | Split Out Slides | Receive Webhook Paylotd and Compile URL's |
| Split Out Slides | n8n-nodes-base.splitOut | Splits `body.slides` into one item per slide | Webhook | Upload media | Receive Webhook Paylotd and Compile URL's |
| Upload media | @blotato/n8n-nodes-blotato.blotato | Uploads each slide image to Blotato media storage | Split Out Slides | Aggregate | Receive Webhook Paylotd and Compile URL's |
| Aggregate | n8n-nodes-base.aggregate | Aggregates uploaded media URLs into an array | Upload media | Caption Generator | Receive Webhook Paylotd and Compile URL's |
| Caption Generator | @n8n/n8n-nodes-langchain.openAi | Generates Instagram caption text with OpenAI | Aggregate | Send message and wait for response | Caption Generate & Approval |
| Send message and wait for response | n8n-nodes-base.telegram | Sends approval request and waits for Telegram response | Caption Generator | If | Caption Generate & Approval |
| If | n8n-nodes-base.if | Routes execution based on approval result | Send message and wait for response | Create post; Send a text message | Approve & Post |
| Create post | @blotato/n8n-nodes-blotato.blotato | Creates/publishes Instagram carousel post via Blotato | If |  | Approve & Post |
| Send a text message | n8n-nodes-base.telegram | Sends rejection/fallback message | If | No Operation, do nothing | Approve & Post |
| No Operation, do nothing | n8n-nodes-base.noOp | Ends the rejection branch explicitly | Send a text message |  | Approve & Post |
| Sticky Note | n8n-nodes-base.stickyNote | Documentation note on workflow purpose, setup, and requirements |  |  | Auto-Create Instagram Carousel Posts from Canva (Webhook → AI Caption → Instagram) |
| Sticky Note1 | n8n-nodes-base.stickyNote | Visual section note for webhook and URL compilation block |  |  | Receive Webhook Paylotd and Compile URL's |
| Sticky Note2 | n8n-nodes-base.stickyNote | Visual section note for caption generation and approval block |  |  | Caption Generate & Approval |
| Sticky Note3 | n8n-nodes-base.stickyNote | Visual section note for approval and posting block |  |  | Approve & Post |
| Sticky Note4 | n8n-nodes-base.stickyNote | External resource note |  |  | - [@youtube](https://www.youtube.com/watch?v=UJcv3qSXpG8) |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it something like: `Claude Canva Files → Blotato`.
   - Leave it inactive until credentials and test payloads are verified.

2. **Add a Webhook node**
   - Node type: `Webhook`
   - HTTP Method: `POST`
   - Path: `canva-pngs-to-drive`
   - Keep default options unless your sender requires custom response behavior.
   - This node will become the workflow trigger.
   - Expected incoming payload shape:
     - It must contain `body.slides`
     - Each slide item should include an `export_url`

3. **Add a Split Out node**
   - Node type: `Split Out`
   - Field to split out: `body.slides`
   - Connect `Webhook → Split Out Slides`
   - Purpose: convert the slides array into one item per slide.

4. **Add a Blotato node for media upload**
   - Node type: `Blotato`
   - Name it `Upload media`
   - Resource: `media`
   - Media URL field: `={{ $json.export_url }}`
   - Connect `Split Out Slides → Upload media`
   - Configure Blotato API credentials in this node.
   - Ensure your upstream slide objects expose a direct image URL at `export_url`.

5. **Add an Aggregate node**
   - Node type: `Aggregate`
   - Configure it to aggregate the field `url`
   - This should collect all uploaded media URLs into one array
   - Connect `Upload media → Aggregate`

6. **Add an OpenAI node**
   - Node type: `OpenAI` from the LangChain/OpenAI nodes
   - Name it `Caption Generator`
   - Select model: `gpt-5-mini`
   - Add response/prompt content equivalent to:
     - `Create an effective caption for Soulfit Pilates. Output only the caption and hashtags without other marks or explanations.`
     - `You're a caption generator agent for Soulfit Pilates. Your job is to come up with Instagram Carousel posts captions to go with the Carousel`
   - Connect `Aggregate → Caption Generator`
   - Configure valid OpenAI credentials.
   - Note: this workflow does not currently pass the image content or slide text into the prompt, so the caption is generic to the brand rather than slide-specific.

7. **Add a Telegram node for approval**
   - Node type: `Telegram`
   - Name it `Send message and wait for response`
   - Operation: `Send and Wait`
   - Chat ID: replace placeholder with the real Telegram chat ID
   - Disable attribution if desired
   - Set approval type to `double`
   - Use a message similar to:
     - `Today's IG Carousel:`
     - `1. {{ $('Aggregate').item.json.url[0] }}`
     - `2. {{ $('Aggregate').item.json.url[1] }}`
     - `3. {{ $('Aggregate').item.json.url[2] }}`
     - `4. {{ $('Aggregate').item.json.url[3] }}`
     - `5. {{ $('Aggregate').item.json.url[4] }}`
     - `IG Caption: {{ $json.output[0].content[0].text }}`
   - Connect `Caption Generator → Send message and wait for response`
   - Configure Telegram bot credentials.
   - Make sure the bot can send messages to the specified chat.

8. **Add an If node**
   - Node type: `If`
   - Condition:
     - Left value: `={{ $json.data.approved }}`
     - Operator: boolean equals
     - Right value: `true`
   - Keep strict comparison enabled if available.
   - Connect `Send message and wait for response → If`

9. **Add a Blotato node for publishing**
   - Node type: `Blotato`
   - Name it `Create post`
   - Connect it to the **true** output of the `If` node
   - Configure:
     - Account ID: replace `REPLACE_WITH_YOUR_ACCOUNT_ID` with your actual Blotato-connected account ID
     - Post content text: `={{ $('Caption Generator').item.json.output[0].content[0].text }}`
     - Post content media URLs: `={{ $('Aggregate').item.json.url }}`
   - Reuse the same Blotato credential as the upload node if appropriate.
   - This node should create the final Instagram carousel post.

10. **Add a Telegram node for rejection handling**
    - Node type: `Telegram`
    - Name it `Send a text message`
    - Connect it to the **false** output of the `If` node
    - Configure:
      - Chat ID: same Telegram target or another review chat
      - Text: `Got it. In that case, please check your Canva files`
      - Disable attribution
    - This serves as a manual-review fallback.

11. **Add a No Operation node**
    - Node type: `No Operation`
    - Name it `No Operation, do nothing`
    - Connect `Send a text message → No Operation, do nothing`
    - This explicitly ends the rejection path.

12. **Optionally add sticky notes for documentation**
    - Add one note describing the overall automation purpose, setup requirements, and workflow summary.
    - Add separate notes over the main sections:
      - `Receive Webhook Paylotd and Compile URL's`
      - `Caption Generate & Approval`
      - `Approve & Post`
    - Add an external resource note with:
      - `[@youtube](https://www.youtube.com/watch?v=UJcv3qSXpG8)`

13. **Configure credentials**
    - **OpenAI credential**
      - Add an OpenAI API key credential in n8n
      - Ensure the selected model is available to your account
    - **Blotato credential**
      - Add the Blotato API credential
      - Confirm the account ID used in `Create post` matches the connected Instagram-capable account
    - **Telegram credential**
      - Create or connect a Telegram bot
      - Obtain the target chat ID
      - Make sure the bot has permission to message that chat

14. **Test with a sample payload**
    - Send a POST request to the webhook test URL with data shaped like:
      - `body.slides` = array
      - each slide object has `export_url`
    - Confirm:
      - split node emits one item per slide
      - upload node returns a `url`
      - aggregate node returns an array `url`
      - caption node returns text at `output[0].content[0].text`
      - Telegram approval returns `data.approved`

15. **Replace all placeholders**
    - Replace both occurrences of `REPLACE_WITH_YOUR_ACCOUNT_ID`
      - Telegram `chatId`
      - Blotato `accountId`
    - If your chat ID and Blotato account ID are different, make sure each field uses the proper value; the placeholder text is reused in the source workflow but refers to different systems.

16. **Activate the workflow**
    - After all tests pass, activate the workflow.
    - Use the production webhook URL in your upstream Canva/Claude/automation system.

## Sub-workflow setup
This workflow does **not** invoke any sub-workflows and is not itself documented as a sub-workflow. There are no `Execute Workflow` nodes or reusable child flows in the provided JSON.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Auto-Create Instagram Carousel Posts from Canva (Webhook → AI Caption → Instagram). Automatically turn Canva carousel exports into a fully published Instagram post with AI-generated captions. Receives webhook payload containing PNG slides, uploads images, generates an optimized Instagram caption using AI, and publishes via Blotato. | Workflow purpose |
| This template removes manual work of downloading slides, writing captions, and posting to Instagram. | Workflow purpose |
| Example automation pipeline: Claude → Canva → n8n → Instagram | Workflow context |
| Requirements listed in the workflow note: OpenAI API credential, Blotato API credential, Instagram account connected to Blotato, and a system that sends the webhook. | Setup requirements |
| Setup note: import the workflow, configure OpenAI and Blotato credentials, copy the webhook URL, connect upstream automation, test with a sample payload, then activate. | Setup guidance |
| YouTube resource | https://www.youtube.com/watch?v=UJcv3qSXpG8 |

## Additional implementation notes
- The workflow title supplied by the user is **“Auto-create Instagram carousel posts from Canva with OpenAI and Blotato”**, while the internal n8n workflow name in JSON is **“Claude Canva Files → Blotato”**.
- The Telegram approval message hardcodes five slide URLs. If the carousel contains fewer or more than five slides, the message preview will be incomplete or contain undefined values, even though the final post may still use the full aggregated media array.
- The OpenAI prompt does not include slide content, so the generated caption is brand-oriented but not context-aware to the actual carousel topic.
- The sticky note text contains a typo: `Receive Webhook Paylotd and Compile URL's`. This has been preserved exactly where relevant.
- The workflow is currently `active: false` in the provided JSON.