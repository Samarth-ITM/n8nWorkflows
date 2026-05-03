Rewrite viral LinkedIn posts in your voice with Google Gemini and Telegram approval

https://n8nworkflows.xyz/workflows/rewrite-viral-linkedin-posts-in-your-voice-with-google-gemini-and-telegram-approval-11882


# Rewrite viral LinkedIn posts in your voice with Google Gemini and Telegram approval

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:**  
This workflow lets you send a LinkedIn post URL to a Telegram bot, automatically **scrape the original post**, **rewrite it in your personal voice using Google Gemini**, **generate an illustrative image**, then **ask for approval in Telegram** before **publishing to LinkedIn**.

**Target use cases:**
- Rewriting high-performing (“viral”) LinkedIn posts into your own consistent style
- Building an approval-based content pipeline (human-in-the-loop) for social publishing
- Automating content + image generation for faster posting

### 1.1 Input Reception & Security
Receives Telegram messages, ensures only your Telegram user can use the bot, extracts a LinkedIn post URL, and validates it.

### 1.2 Scraping & Persona Loading
Scrapes the LinkedIn post via ConnectSafely.ai, extracts key fields, and loads a customizable “persona” object representing your writing style.

### 1.3 AI Rewrite (Gemini) + Structured Output
Gemini rewrites the post in your voice. A structured output parser enforces valid JSON output with a single `post` field.

### 1.4 Image Prompting + Image Generation + Telegram Preview
Builds an image prompt based on the rewritten post, generates an image using Gemini image generation, and sends the preview image to Telegram.

### 1.5 Approval Gate
Sends the generated text to Telegram with “approve/decline” buttons and waits for your response.

### 1.6 Publish to LinkedIn + Confirmation
If approved, cleans formatting, attaches the generated image binary, publishes to LinkedIn, and sends a success message; otherwise sends a decline message.

---

## 2. Block-by-Block Analysis

### Block 1 — Input Reception & Validation
**Overview:**  
Listens for Telegram messages, blocks unauthorized users, extracts the LinkedIn URL from the message, and routes invalid input to an error response.

**Nodes involved:**
- Telegram Trigger
- 🔒 Security Check
- 🔍 Extract Post URL
- ❓ Valid URL?
- Send Error Message

#### Node: **Telegram Trigger**
- **Type / Role:** `telegramTrigger` — entry point; receives incoming Telegram updates.
- **Configuration choices:**
  - Subscribed to `message` updates.
- **Key variables:**
  - Incoming payload available at `$json.message...` (notably `message.text`, `message.from.id`).
- **Connections:**
  - Output → **🔒 Security Check**
- **Failure/edge cases:**
  - Telegram credentials/webhook misconfigured; webhook not reachable from n8n.
  - Non-text messages (photos/stickers) may not have `message.text`.

#### Node: **🔒 Security Check**
- **Type / Role:** `if` — restricts workflow usage to a single Telegram user ID.
- **Configuration choices:**
  - Condition: `{{$json.message.from.id}} equals YOUR_TELEGRAM_USER_ID`
- **Key variables/expressions:**
  - `leftValue`: `={{ $json.message.from.id }}`
  - `rightValue`: `"YOUR_TELEGRAM_USER_ID"` (placeholder)
- **Connections:**
  - True → **🔍 Extract Post URL**
  - False → (no outgoing connection defined; unauthorized messages effectively stop here)
- **Version notes:** If node type v2.2 uses strict type validation; ensure ID is numeric.
- **Failure/edge cases:**
  - If `YOUR_TELEGRAM_USER_ID` not replaced, nobody will pass the check.
  - Group chats may have different sender contexts; still uses `message.from.id`.

#### Node: **🔍 Extract Post URL**
- **Type / Role:** `code` — extracts the first LinkedIn post URL from the Telegram message text.
- **Configuration choices:**
  - Regex: `/(https?:\/\/(?:www\.)?linkedin\.com\/posts\/[^\s]+)/gi`
  - Returns `{ valid: false, message: "...", originalInput }` if none found
  - Returns `{ postUrl, valid: true, workflowId: $execution.id, extractedAt, originalInput }` if found
- **Key variables:**
  - Reads: `$input.first().json.message.text`
  - Writes: `postUrl`, `valid`, etc.
- **Connections:**
  - Output → **❓ Valid URL?**
- **Failure/edge cases:**
  - Only matches URLs under `linkedin.com/posts/...`; it will **not** match other LinkedIn formats (e.g., `linkedin.com/feed/update/urn:li:activity:...`).
  - If user sends multiple URLs, only the first is used.

#### Node: **❓ Valid URL?**
- **Type / Role:** `if` — routes based on extracted URL validity.
- **Configuration choices:**
  - Condition: `={{ $json.valid }}` is true.
- **Connections:**
  - True → **🔍 Scrape LinkedIn Post**
  - False → **Send Error Message**
- **Failure/edge cases:**
  - If upstream code changes field names, this gate breaks.

#### Node: **Send Error Message**
- **Type / Role:** `telegram` — sends validation failure message back to user.
- **Configuration choices:**
  - Text: `={{ $('🔍 Extract Post URL').item.json.message }}`
  - chatId: `={{ $('Telegram Trigger').item.json.message.from.id }}`
- **Connections:** none
- **Failure/edge cases:**
  - If node `🔍 Extract Post URL` does not produce `message` (e.g., unexpected code path), expression will fail.
  - Telegram auth errors.

---

### Block 2 — Scrape & Persona
**Overview:**  
Scrapes the LinkedIn post content via ConnectSafely.ai, extracts key fields (text, author, likes, media flags), then injects your persona definition for style-matching.

**Nodes involved:**
- 🔍 Scrape LinkedIn Post
- 📋 Extract Post Data
- 👤 Load Your Persona

#### Node: **🔍 Scrape LinkedIn Post**
- **Type / Role:** `connectSafelyLinkedIn` — external scraping integration for LinkedIn post content.
- **Configuration choices:**
  - Operation: `scrapePost`
  - `postUrl`: `={{ $json.postUrl }}`
- **Connections:**
  - Output → **📋 Extract Post Data**
- **Failure/edge cases:**
  - ConnectSafely.ai credentials/limits; scraping may fail due to LinkedIn restrictions.
  - Post URL not accessible (private post, deleted post).
  - Response shape changes could break downstream set expressions.

#### Node: **📋 Extract Post Data**
- **Type / Role:** `set` — maps scraped response into simpler fields for later prompts.
- **Configuration choices (assignments):**
  - `postText` ← `={{ $json.data.content }}`
  - `authorName` ← `={{ $json.data.author.name }}`
  - `engagement` ← `={{ $json.data.engagement.likes }}`
  - `hasImages` ← `={{ $json.data.media.hasImages }}`
- **Connections:**
  - Output → **👤 Load Your Persona**
- **Failure/edge cases:**
  - If scrape output is missing `data.content` or `data.author.name`, expressions may resolve to `null` and degrade rewrite quality.
  - `engagement.likes` may be non-numeric or absent.

#### Node: **👤 Load Your Persona**
- **Type / Role:** `code` — defines and injects a `persona` JSON object describing your writing style.
- **Configuration choices:**
  - Persona object includes: `name`, `style` (tone/voice/formatting/hooks/CTA), `expertiseAreas`, `commonPhrases`, `preferredEmojis`, `postLength`, etc.
  - Merges persona + scraped fields into one JSON.
- **Key variables:**
  - Reads: `$input.first().json.postText`, `authorName`, `engagement`
  - Writes: `persona`, `originalPostText`, `originalAuthor`, `originalEngagement`, `personaLoaded`
- **Connections:**
  - Output → **AI Agent1**
- **Failure/edge cases:**
  - Persona arrays referenced later (`persona.expertiseAreas.join(...)`) must exist; removing them without updating prompt can break.
  - Very long scraped text could increase token usage in Gemini.

---

### Block 3 — AI Rewrite (Gemini) + Structured Output
**Overview:**  
Uses a Gemini chat model to rewrite the scraped post to match the persona. A structured parser enforces the JSON output schema containing the final `post`.

**Nodes involved:**
- Google Gemini Chat Model
- Structured Output Parser
- AI Agent1

#### Node: **Google Gemini Chat Model**
- **Type / Role:** `lmChatGoogleGemini` — LLM backend for the agent.
- **Configuration choices:**
  - Model: `models/gemini-2.5-pro`
  - Temperature: `1` (more creative variability)
- **Connections:**
  - AI output (language model port) → **AI Agent1**
- **Failure/edge cases:**
  - Missing/invalid Google credentials, model access not enabled, quota exceeded.
  - Higher temperature increases chance of format drift (mitigated by parser).

#### Node: **Structured Output Parser**
- **Type / Role:** `outputParserStructured` — validates and parses model output into a strict JSON structure.
- **Configuration choices:**
  - Manual JSON schema requiring:
    - `post` (string, minLength 100, maxLength 3000)
  - `additionalProperties: false`
- **Connections:**
  - Parser port → **AI Agent1**
- **Failure/edge cases:**
  - If Gemini outputs non-JSON or violates schema, the agent node will error/fail.
  - Post shorter than 100 chars will be rejected.

#### Node: **AI Agent1**
- **Type / Role:** `langchain.agent` — orchestrates prompt + model + output parser to produce final rewritten post.
- **Configuration choices:**
  - User instruction: “Rewrite the post now.”
  - System message: large prompt that:
    - Includes original author/engagement/text
    - Includes persona details and formatting rules
    - Requires output to be **ONLY JSON** with `{ "post": "..." }`
  - `hasOutputParser: true` (connected to Structured Output Parser)
- **Key expressions used inside prompt:**
  - Original fields: `{{ $json.originalAuthor }}`, `{{ $json.originalEngagement }}`, `{{ $json.originalPostText }}`
  - Persona fields (many): e.g. `{{ $json.persona.expertiseAreas.join(', ') }}`
- **Connections:**
  - Main output → **📝 Create Image Prompt**
- **Failure/edge cases:**
  - If persona fields are missing, prompt interpolation may fail or render “undefined”.
  - Output may still fail schema if too long/short or contains invalid JSON escaping.

---

### Block 4 — Image Prompt + Image Generation + Telegram Preview
**Overview:**  
Creates an image prompt from the rewritten post, generates an image (binary) with Gemini image generation, and sends it as a Telegram photo preview.

**Nodes involved:**
- 📝 Create Image Prompt
- Generate an image
- 📨 Send Preview Image

#### Node: **📝 Create Image Prompt**
- **Type / Role:** `code` — derives an illustration prompt from the rewritten post.
- **Configuration choices:**
  - Takes first 300 chars of the post for context.
  - Enforces “No text in the image.”
- **Key variables:**
  - Reads: `$input.first().json.output.post` (from AI Agent1)
  - Writes: `{ imagePrompt, post }`
- **Connections:**
  - Output → **Generate an image**
- **Failure/edge cases:**
  - If AI output path changes (e.g., not `json.output.post`), prompt generation fails.

#### Node: **Generate an image**
- **Type / Role:** `googleGemini` (LangChain integration) — generates an image from a prompt and returns binary data.
- **Configuration choices:**
  - Resource: `image`
  - Model: `models/gemini-2.0-flash-exp-image-generation`
  - Output binary property: `data`
- **Connections:**
  - Output → **📨 Send Preview Image**
- **Failure/edge cases:**
  - Model availability/permissions vary by region/account.
  - Prompt may be rejected by provider policy; node returns error.
  - Large images may exceed Telegram limits depending on settings.

#### Node: **📨 Send Preview Image**
- **Type / Role:** `telegram` — sends generated image to Telegram chat.
- **Configuration choices:**
  - Operation: `sendPhoto`
  - `binaryData: true` (expects binary on item)
  - Caption: “🎨 Generated Image for your post (Gemini)”
  - chatId: `={{ $('Telegram Trigger').item.json.message.from.id }}`
- **Connections:**
  - Output → **Send Message and Wait for Approval**
- **Failure/edge cases:**
  - If binary property isn’t on the current item (or name differs), sendPhoto fails.
  - Telegram file size constraints.

---

### Block 5 — Approval Flow
**Overview:**  
Sends the rewritten text to Telegram and waits for explicit approval/rejection. Routes to publish or decline accordingly.

**Nodes involved:**
- Send Message and Wait for Approval
- Check Approval
- Send Decline Message

#### Node: **Send Message and Wait for Approval**
- **Type / Role:** `telegram` — interactive approval gate using “sendAndWait”.
- **Configuration choices:**
  - Operation: `sendAndWait`
  - Approval type: `double` (yes/no)
  - Message includes rewritten text from AI Agent:
    - `{{ $('AI Agent1').first().json.output.post }}`
  - chatId: from trigger sender
- **Connections:**
  - Output → **Check Approval**
- **Failure/edge cases:**
  - If user never responds, execution waits until node timeout (depends on n8n settings).
  - If message content exceeds Telegram limits, send fails.
  - Expression depends on `AI Agent1` node output existing.

#### Node: **Check Approval**
- **Type / Role:** `if` — branches on approval result.
- **Configuration choices:**
  - Condition: `={{ $json.data.approved }}` is true
- **Connections:**
  - True → **Prepare Content for LinkedIn**
  - False → **Send Decline Message**
- **Failure/edge cases:**
  - If Telegram node output schema changes and `data.approved` is absent, branch may misbehave.

#### Node: **Send Decline Message**
- **Type / Role:** `telegram` — informs user that post won’t be published.
- **Configuration choices:**
  - Text: “Your post was not published. You decided to decline.”
  - chatId: trigger sender
- **Connections:** none
- **Failure/edge cases:** Telegram auth/permissions.

---

### Block 6 — Publish to LinkedIn + Confirmation
**Overview:**  
Cleans the AI-generated LinkedIn formatting, attaches the generated image binary, creates an image post on LinkedIn, then sends a success message with the resulting LinkedIn URL.

**Nodes involved:**
- Prepare Content for LinkedIn
- Create LinkedIn Post
- Send Success Message

#### Node: **Prepare Content for LinkedIn**
- **Type / Role:** `code` — normalizes post text and forwards image binary for LinkedIn upload.
- **Configuration choices:**
  - Reads `$('AI Agent1').first().json.output.post`
  - Cleans:
    - Converts escaped `\\n` to actual newlines
    - Strips `**bold**` and `*bold*` markers (note: this conflicts with prompt suggesting `*bold*` for LinkedIn; this node removes those markers)
    - Removes markdown headers
    - Limits excessive newlines
    - Ensures hashtags are on last line
  - Pulls image binary: `$('Generate an image').first().binary.data`
  - Outputs:
    - `json.post` cleaned
    - `binary.data` image
- **Connections:**
  - Output → **Create LinkedIn Post**
- **Failure/edge cases:**
  - If image generation failed or binary missing, `binary.data` access throws.
  - Hashtag regex assumes `#\w+` (won’t match hashtags with non-latin chars or punctuation).
  - Removes asterisks that you might want to keep for emphasis.

#### Node: **Create LinkedIn Post**
- **Type / Role:** `linkedIn` — publishes an IMAGE post.
- **Configuration choices:**
  - Text: `={{ $json.post }}`
  - Person: `YOUR_LINKEDIN_PERSON_ID` (placeholder; must be replaced)
  - `shareMediaCategory: IMAGE`
  - `binaryPropertyName: data` (must match binary key from previous node)
- **Connections:**
  - Output → **Send Success Message**
- **Failure/edge cases:**
  - LinkedIn OAuth not configured or expired.
  - Incorrect `person` URN/ID format.
  - LinkedIn API may reject text length, media upload issues, rate limits.

#### Node: **Send Success Message**
- **Type / Role:** `telegram` — sends a “post is live” message with a LinkedIn link.
- **Configuration choices:**
  - Text includes: `https://www.linkedin.com/feed/update/{{ $json.urn }}`
  - chatId: trigger sender
- **Connections:** none
- **Failure/edge cases:**
  - If LinkedIn node does not return `urn`, link generation fails.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| 📋 Workflow Overview | Sticky Note | Documentation / overview | — | — | @[youtube](urteFD1Q-74)<br>LinkedIn Viral Post Creator — Telegram Bot → AI Rewrite → Auto-Post<br>Setup Required: Telegram Bot, Google Gemini API, ConnectSafely.ai, LinkedIn OAuth, Your Telegram ID, Your Persona |
| 📥 Input & Validation | Sticky Note | Block label | — | — | Receives Telegram messages, validates sender, and extracts LinkedIn URLs. |
| Telegram Trigger | telegramTrigger | Entry point: receive Telegram message | — | 🔒 Security Check | Receives Telegram messages, validates sender, and extracts LinkedIn URLs. |
| 🔒 Security Check | if | Allow only your Telegram user ID | Telegram Trigger | 🔍 Extract Post URL (true) | Receives Telegram messages, validates sender, and extracts LinkedIn URLs. |
| 🔍 Extract Post URL | code | Extract LinkedIn post URL from text | 🔒 Security Check | ❓ Valid URL? | Receives Telegram messages, validates sender, and extracts LinkedIn URLs. |
| ❓ Valid URL? | if | Route valid vs invalid extraction | 🔍 Extract Post URL | 🔍 Scrape LinkedIn Post (true), Send Error Message (false) | Receives Telegram messages, validates sender, and extracts LinkedIn URLs. |
| Send Error Message | telegram | Send validation error to Telegram | ❓ Valid URL? (false) | — | Receives Telegram messages, validates sender, and extracts LinkedIn URLs. |
| 🔄 Scrape & Persona | Sticky Note | Block label | — | — | Scrapes original post content and loads your personal writing style. |
| 🔍 Scrape LinkedIn Post | connectSafelyLinkedIn | Scrape post content via ConnectSafely.ai | ❓ Valid URL? (true) | 📋 Extract Post Data | Scrapes original post content and loads your personal writing style. |
| 📋 Extract Post Data | set | Map scraped fields (text/author/likes/media) | 🔍 Scrape LinkedIn Post | 👤 Load Your Persona | Scrapes original post content and loads your personal writing style. |
| 👤 Load Your Persona | code | Inject persona object & normalize fields | 📋 Extract Post Data | AI Agent1 | Scrapes original post content and loads your personal writing style. |
| 🤖 AI Generation | Sticky Note | Block label | — | — | Rewrites post in your voice using Gemini, then creates an image prompt. |
| Google Gemini Chat Model | lmChatGoogleGemini | LLM backend (Gemini) | — | AI Agent1 (ai_languageModel) | Rewrites post in your voice using Gemini, then creates an image prompt. |
| Structured Output Parser | outputParserStructured | Enforce JSON schema `{post: string}` | — | AI Agent1 (ai_outputParser) | Rewrites post in your voice using Gemini, then creates an image prompt. |
| AI Agent1 | langchain.agent | Rewrite scraped post in persona style | 👤 Load Your Persona | 📝 Create Image Prompt | Rewrites post in your voice using Gemini, then creates an image prompt. |
| 📝 Create Image Prompt | code | Generate image prompt from rewritten post | AI Agent1 | Generate an image | Rewrites post in your voice using Gemini, then creates an image prompt. |
| 🎨 Image & Preview | Sticky Note | Block label | — | — | Generates image with Gemini Imagen and sends preview to Telegram. |
| Generate an image | googleGemini (image) | Generate image binary from prompt | 📝 Create Image Prompt | 📨 Send Preview Image | Generates image with Gemini Imagen and sends preview to Telegram. |
| 📨 Send Preview Image | telegram | Send generated image to Telegram | Generate an image | Send Message and Wait for Approval | Generates image with Gemini Imagen and sends preview to Telegram. |
| ✅ Approval Flow | Sticky Note | Block label | — | — | Waits for your yes/no response in Telegram before proceeding. |
| Send Message and Wait for Approval | telegram | Send text + wait for yes/no | 📨 Send Preview Image | Check Approval | Waits for your yes/no response in Telegram before proceeding. |
| Check Approval | if | Branch on approved/declined | Send Message and Wait for Approval | Prepare Content for LinkedIn (true), Send Decline Message (false) | Waits for your yes/no response in Telegram before proceeding. |
| Send Decline Message | telegram | Notify decline | Check Approval (false) | — | Waits for your yes/no response in Telegram before proceeding. |
| 🚀 Publish | Sticky Note | Block label | — | — | Posts to LinkedIn with image and sends confirmation to Telegram. |
| Prepare Content for LinkedIn | code | Clean post + attach image binary for LinkedIn | Check Approval (true) | Create LinkedIn Post | Posts to LinkedIn with image and sends confirmation to Telegram. |
| Create LinkedIn Post | linkedIn | Publish IMAGE post to LinkedIn | Prepare Content for LinkedIn | Send Success Message | Posts to LinkedIn with image and sends confirmation to Telegram. |
| Send Success Message | telegram | Send live link confirmation | Create LinkedIn Post | — | Posts to LinkedIn with image and sends confirmation to Telegram. |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.
2. **Add “Telegram Trigger” (Telegram Trigger node)**
   - Updates: `message`
   - Configure Telegram credentials (bot token).
3. **Add IF node “🔒 Security Check”**
   - Condition: Number → Equals  
     - Left: `{{$json.message.from.id}}`  
     - Right: your numeric Telegram user ID (get from `@userinfobot`)
   - Connect: Telegram Trigger → 🔒 Security Check (main).
4. **Add Code node “🔍 Extract Post URL”**
   - Paste code that:
     - Reads `message.text`
     - Regex matches `linkedin.com/posts/...`
     - Outputs `{valid:false,message:"..."}` or `{valid:true,postUrl:"..."}`
   - Connect: 🔒 Security Check (true) → 🔍 Extract Post URL.
5. **Add IF node “❓ Valid URL?”**
   - Condition: Boolean → is true  
     - Left: `{{$json.valid}}`
   - Connect: 🔍 Extract Post URL → ❓ Valid URL?
6. **Add Telegram node “Send Error Message”**
   - Operation: Send Message
   - chatId: `{{ $('Telegram Trigger').item.json.message.from.id }}`
   - Text: `{{ $('🔍 Extract Post URL').item.json.message }}`
   - Connect: ❓ Valid URL? (false) → Send Error Message.
7. **Add ConnectSafely.ai LinkedIn node “🔍 Scrape LinkedIn Post”**
   - Operation: `scrapePost`
   - postUrl: `{{$json.postUrl}}`
   - Configure ConnectSafely.ai credentials.
   - Connect: ❓ Valid URL? (true) → 🔍 Scrape LinkedIn Post.
8. **Add Set node “📋 Extract Post Data”**
   - Add fields:
     - `postText` = `{{$json.data.content}}`
     - `authorName` = `{{$json.data.author.name}}`
     - `engagement` = `{{$json.data.engagement.likes}}`
     - `hasImages` = `{{$json.data.media.hasImages}}`
   - Connect: 🔍 Scrape LinkedIn Post → 📋 Extract Post Data.
9. **Add Code node “👤 Load Your Persona”**
   - Create a `PERSONA` object and return merged JSON including:
     - `persona`, `originalPostText`, `originalAuthor`, `originalEngagement`
   - Customize persona fields to match your voice.
   - Connect: 📋 Extract Post Data → 👤 Load Your Persona.
10. **Add Google Gemini Chat Model node “Google Gemini Chat Model”**
    - Model: `models/gemini-2.5-pro`
    - Temperature: `1`
    - Configure Google Gemini / Google AI Studio credentials.
11. **Add Structured Output Parser node “Structured Output Parser”**
    - Schema: object with required `post` (string 100–3000), no additional props.
12. **Add AI Agent node “AI Agent1”**
    - System message: include original post + persona fields + strict JSON output requirement.
    - Enable output parsing and connect:
      - Google Gemini Chat Model → AI Agent1 (language model port)
      - Structured Output Parser → AI Agent1 (output parser port)
    - Connect: 👤 Load Your Persona → AI Agent1 (main).
13. **Add Code node “📝 Create Image Prompt”**
    - Build `imagePrompt` from `AI Agent1` output post (e.g., first ~300 chars).
    - Output `{imagePrompt, post}`.
    - Connect: AI Agent1 → 📝 Create Image Prompt.
14. **Add Gemini Image Generation node “Generate an image”**
    - Resource: Image
    - Model: `models/gemini-2.0-flash-exp-image-generation`
    - Binary output property: `data`
    - Connect: 📝 Create Image Prompt → Generate an image.
15. **Add Telegram node “📨 Send Preview Image”**
    - Operation: Send Photo
    - Binary data: enabled (expects `binary.data`)
    - Caption as desired
    - chatId: `{{ $('Telegram Trigger').item.json.message.from.id }}`
    - Connect: Generate an image → 📨 Send Preview Image.
16. **Add Telegram node “Send Message and Wait for Approval”**
    - Operation: `sendAndWait`
    - Approval options: “double” (approve/decline)
    - Message text includes the rewritten post:
      - `{{ $('AI Agent1').first().json.output.post }}`
    - chatId: from trigger sender
    - Connect: 📨 Send Preview Image → Send Message and Wait for Approval.
17. **Add IF node “Check Approval”**
    - Condition: Boolean true on `{{$json.data.approved}}`
    - Connect: Send Message and Wait for Approval → Check Approval.
18. **Add Telegram node “Send Decline Message”**
    - Send a simple decline notice to same chatId.
    - Connect: Check Approval (false) → Send Decline Message.
19. **Add Code node “Prepare Content for LinkedIn”**
    - Clean text (newline fixes, hashtag last line)
    - Attach `binary.data` from “Generate an image”
    - Output item with `json.post` and `binary.data`
    - Connect: Check Approval (true) → Prepare Content for LinkedIn.
20. **Add LinkedIn node “Create LinkedIn Post”**
    - Operation: create share/post (image)
    - Text: `{{$json.post}}`
    - Share media category: IMAGE
    - Binary property name: `data`
    - Person: set your LinkedIn Person ID (replace placeholder)
    - Configure LinkedIn OAuth2 credentials in n8n.
    - Connect: Prepare Content for LinkedIn → Create LinkedIn Post.
21. **Add Telegram node “Send Success Message”**
    - chatId: trigger sender
    - Text includes LinkedIn URL using returned `urn`:
      - `https://www.linkedin.com/feed/update/{{ $json.urn }}`
    - Connect: Create LinkedIn Post → Send Success Message.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| YouTube video reference embedded in the workflow note | @[youtube](urteFD1Q-74) |
| Replace `YOUR_TELEGRAM_USER_ID` with your actual Telegram user ID (e.g., from @userinfobot) | Security gating in **🔒 Security Check** |
| Replace `YOUR_LINKEDIN_PERSON_ID` with your LinkedIn Person ID | Publishing in **Create LinkedIn Post** |
| ConnectSafely.ai is required for LinkedIn scraping | Node **🔍 Scrape LinkedIn Post** |
| Persona must be customized to match your voice | Node **👤 Load Your Persona** notes: “CUSTOMIZE THIS” |