Translate multilingual content with DeepL, GPT-4, WordPress, Slack and Sheets

https://n8nworkflows.xyz/workflows/translate-multilingual-content-with-deepl--gpt-4--wordpress--slack-and-sheets-11910


# Translate multilingual content with DeepL, GPT-4, WordPress, Slack and Sheets

## 1. Workflow Overview

**Workflow name:** Multi-Language Content Translation Pipeline with AI Quality Control  
**Stated title:** Translate multilingual content with DeepL, GPT-4, WordPress, Slack and Sheets

**Purpose:**  
This workflow receives source content (via **Webhook** or **Schedule**), translates it into multiple target languages using **DeepL**, then uses an **OpenAI (LangChain) AI agent** to score translation quality. Approved translations are saved as **WordPress drafts**; non-approved translations are **flagged in Slack** for manual review. Finally, all translation results are **aggregated**, **logged to Google Sheets** (translation memory), and a **Slack summary** is sent. If triggered by Webhook, the workflow returns a JSON response.

### 1.1 Trigger & Routing
Two entry points (Webhook and Schedule) converge into a single flow.

### 1.2 Configuration & Source Normalization
Defines source/target languages and normalizes incoming content into a consistent internal schema.

### 1.3 Per-Language Translation
Splits target languages into individual items and calls DeepL.

### 1.4 AI Quality Guard
An OpenAI-based agent evaluates each translation and outputs a structured QA result.

### 1.5 Distribution, Logging, and Completion
Branches on approval: publish as WordPress draft vs Slack flag. Aggregates all items, appends to Google Sheets, posts Slack summary, and responds to webhook.

---

## 2. Block-by-Block Analysis

### Block 1 — Trigger & Intake

**Overview:**  
Starts the workflow either on an inbound HTTP request or on a timer. Both routes merge into a single execution path.

**Nodes involved:**  
- Content Webhook  
- Scheduled Batch  
- Merge Triggers

#### Node: Content Webhook
- **Type / role:** Webhook trigger (HTTP entry point).
- **Configuration (interpreted):**
  - Method: **POST**
  - Path: **/translate-content**
  - Response mode: **Respond via “Respond to Webhook” node** (responseNode).
  - **onError:** `continueRegularOutput` (workflow continues even if webhook-related error occurs; may still produce output).
- **Inputs / outputs:**
  - **Output →** Merge Triggers (branch index 0)
- **Key data expectations:**
  - Expects fields like `content` or `text`, optional `title`, optional `id`.
- **Failure modes / edge cases:**
  - Missing/invalid body: downstream “Prepare Source Content” falls back to sample text/title.
  - If the workflow doesn’t reach “Respond to Webhook” (e.g., mid-workflow error), the webhook call may time out client-side.

#### Node: Scheduled Batch
- **Type / role:** Schedule Trigger (time-based entry point).
- **Configuration:**
  - Runs **every 4 hours**.
- **Outputs:**
  - **Output →** Merge Triggers (branch index 1)
- **Failure modes / edge cases:**
  - Scheduled executions have no natural HTTP response; however, the workflow still ends in “Respond to Webhook”, which may be unnecessary/no-op depending on n8n behavior and configuration.

#### Node: Merge Triggers
- **Type / role:** Merge node used for “either/or” trigger routing.
- **Configuration:**
  - Mode: **Choose Branch** (passes through whichever input fires).
- **Inputs / outputs:**
  - **Inputs:** Content Webhook, Scheduled Batch
  - **Output →** Translation Configuration
- **Failure modes / edge cases:**
  - If both triggers fired simultaneously (rare), Choose Branch behavior may select one branch per execution; each execution should still process independently.

---

### Block 2 — Configuration & Source Preparation

**Overview:**  
Defines translation parameters and normalizes the incoming payload into a consistent data structure used throughout the pipeline.

**Nodes involved:**  
- Translation Configuration  
- Prepare Source Content  
- Split Target Languages

#### Node: Translation Configuration
- **Type / role:** Set node (static config).
- **Configuration choices:**
  - Mode: **Raw JSON output** producing:
    - `sourceLanguage`: `"EN"`
    - `targetLanguages`: `["DE","FR","ES","JA"]`
    - `contentType`: `"article"`
    - `preserveFormatting`: `true`
    - `qualityThreshold`: `0.85`
    - `distributionTargets`: `["wordpress","contentful"]` (note: only WordPress is used in this workflow)
- **Inputs / outputs:**
  - **Input:** Merge Triggers
  - **Output →** Prepare Source Content
- **Edge cases:**
  - `qualityThreshold` is **not actually referenced** in the IF logic; the agent prompt hardcodes `>= 0.85`. Changing threshold here currently won’t affect approval unless you update the agent prompt and/or IF logic.

#### Node: Prepare Source Content
- **Type / role:** Code node (data normalization).
- **Configuration choices (logic):**
  - Reads trigger item: `$input.first()`
  - Reads config: `$('Translation Configuration').first().json`
  - Determines:
    - `sourceText` = `item.json.content` OR `item.json.text` OR fallback `"Sample content for translation"`
    - `title` = `item.json.title` OR `"Untitled"`
    - `contentId` = `item.json.id` OR `content-${Date.now()}`
  - Outputs a single normalized item with:
    - `contentId`, `title`, `sourceText`, `sourceLanguage`, `targetLanguages`, `config`, `createdAt`
- **Inputs / outputs:**
  - **Input:** Translation Configuration
  - **Output →** Split Target Languages
- **Edge cases / failure types:**
  - If incoming webhook payload is not JSON or fields are nested differently, `sourceText` may default to sample text unintentionally.
  - `Date.now()`-based IDs can collide in very high concurrency scenarios (unlikely).

#### Node: Split Target Languages
- **Type / role:** SplitOut node (fan-out per language).
- **Configuration:**
  - Field to split out: `targetLanguages`
  - Produces one item per language; each item’s `$json` becomes the target language code (e.g., `"DE"`).
- **Inputs / outputs:**
  - **Input:** Prepare Source Content
  - **Output →** DeepL Translate
- **Edge cases:**
  - If `targetLanguages` is empty or missing, nothing proceeds (no translations).
  - If language codes are invalid for DeepL, DeepL node will error.

---

### Block 3 — DeepL Translation

**Overview:**  
Calls DeepL to translate the normalized source text into each target language.

**Nodes involved:**  
- DeepL Translate  
- Format Translation Result

#### Node: DeepL Translate
- **Type / role:** DeepL API translation node.
- **Configuration choices:**
  - Text: `$('Prepare Source Content').first().json.sourceText`
  - Translate to: `{{ $json }}` (the per-item language code emitted by SplitOut)
- **Inputs / outputs:**
  - **Input:** Split Target Languages
  - **Output →** Format Translation Result
- **Credential requirements:**
  - DeepL API credentials must be configured in n8n.
- **Edge cases / failure types:**
  - Auth/key errors, quota limits, invalid language code.
  - Large text can exceed API limits; current workflow does not chunk long content.
  - Formatting preservation is configured in “Translation Configuration” but not passed into DeepL node parameters here (so it may not actually preserve formatting unless DeepL node defaults do).

#### Node: Format Translation Result
- **Type / role:** Code node (shapes DeepL output + enriches metadata).
- **Configuration choices (logic):**
  - Gets source data from `$('Prepare Source Content').first().json`
  - Gets target language from `$('Split Target Languages').item.json`
  - Extracts translated text from either:
    - `item.json.text` or
    - `item.json.translations?.[0]?.text` or `''`
  - Outputs:
    - `contentId`, `title`, `sourceLanguage`, `targetLanguage`, `sourceText`, `translatedText`,
    - `characterCount`, `translatedAt`
- **Inputs / outputs:**
  - **Input:** DeepL Translate
  - **Output →** AI Quality Reviewer
- **Edge cases:**
  - If DeepL response schema differs, `translatedText` may become empty string and still proceed to QA.
  - Uses `.item` selector on `Split Target Languages` which depends on item pairing; if execution merges/branches unexpectedly, item linkage can break.

---

### Block 4 — AI Quality Guard (LangChain)

**Overview:**  
An AI agent reviews each translation and returns a JSON QA report. The workflow then parses the report and decides whether to approve.

**Nodes involved:**  
- OpenAI Chat Model  
- AI Quality Reviewer  
- Parse Quality Result  
- Quality Approved?

#### Node: OpenAI Chat Model
- **Type / role:** LangChain Chat Model (LLM provider for the agent).
- **Configuration:**
  - Model: `gpt-4o-mini`
  - Temperature: `0.2` (more deterministic evaluations)
- **Connections:**
  - **AI output (language model port) →** AI Quality Reviewer `ai_languageModel`
- **Credential requirements:**
  - OpenAI (or compatible) API credentials configured for the LangChain OpenAI node.
- **Edge cases / failures:**
  - Model availability, rate limits, authentication errors.
  - Temperature low but not zero; occasional formatting drift is still possible.

#### Node: AI Quality Reviewer
- **Type / role:** LangChain Agent node (LLM-driven evaluator).
- **Configuration choices:**
  - Prompt includes:
    - Source language, target language
    - First 500 characters of source and translation (`substring(0, 500)`)
  - Requires output “in this exact JSON format” with scores and approval boolean.
  - System message sets persona: “professional translator and QA specialist”.
  - The approval criterion in prompt: `approved: true if qualityScore >= 0.85` (**hardcoded**).
- **Inputs / outputs:**
  - **Main input:** Format Translation Result
  - **AI language model input:** OpenAI Chat Model
  - **Main output →** Parse Quality Result
- **Edge cases:**
  - Only first 500 chars are evaluated; long articles may be approved despite later issues.
  - Agent may output non-JSON or extra text; parsing node attempts to recover but can fail.

#### Node: Parse Quality Result
- **Type / role:** Code node (robust-ish JSON extraction + fallback).
- **Configuration choices (logic):**
  - Reads translation data from `$('Format Translation Result').item.json`
  - Extracts model response from `item.json.output` or `item.json.text`
  - Regex match: `/\{[\s\S]*\}/` to find a JSON object in the text, then `JSON.parse`
  - Fallback if no JSON match:
    - qualityScore/accuracy/fluency = 0.8, approved = true, summary indicates default approval
  - Catch parsing errors:
    - qualityScore/accuracy/fluency = 0.7, approved = false, issues include “Parse error”
  - Outputs combined object:
    - original translation fields +
    - `quality` (object), `isApproved` (boolean), `processedAt`
- **Inputs / outputs:**
  - **Input:** AI Quality Reviewer
  - **Output →** Quality Approved?
- **Edge cases / failure types:**
  - Regex can capture too much if the agent returns multiple JSON blocks or braces in text.
  - The “no JSON match” fallback **approves by default**, which is risky (it contradicts strict QA intent).

#### Node: Quality Approved?
- **Type / role:** IF node (branching decision).
- **Configuration:**
  - Condition: `{{ $json.isApproved }} == true` (boolean equals).
  - Strict type validation enabled.
- **Inputs / outputs:**
  - **Input:** Parse Quality Result
  - **Output (true) →** Publish to WordPress
  - **Output (false) →** Flag for Manual Review
- **Edge cases:**
  - If `isApproved` is missing or not boolean, strict validation may treat it as non-matching and route to “false” (depending on n8n behavior).

---

### Block 5 — Distribution, Aggregation, Logging, Completion

**Overview:**  
Approved translations are created as WordPress drafts; non-approved ones are posted to Slack for review. All results are aggregated, logged to Google Sheets, summarized in Slack, and a webhook response is returned.

**Nodes involved:**  
- Publish to WordPress  
- Flag for Manual Review  
- Aggregate All Translations  
- Log to Translation Memory  
- Send Completion Summary  
- Respond to Webhook

#### Node: Publish to WordPress
- **Type / role:** WordPress node (content publishing).
- **Configuration:**
  - Creates a post/page (operation not explicitly shown in JSON; defaults apply per node)
  - Title: `{{ $json.title }} ({{ $json.targetLanguage }})`
  - Status: `draft`
- **Inputs / outputs:**
  - **Input:** Quality Approved? (true branch)
  - **Output →** Aggregate All Translations
- **Credential requirements:**
  - WordPress credentials (typically API user/app password or OAuth depending on node setup).
- **Edge cases / failures:**
  - Missing required WP fields (content/body is not mapped here).
  - Auth failures, insufficient permissions, site URL misconfig.
  - This node sets title/status but **does not pass translated content** into WP body in the shown parameters; results may be empty drafts unless defaults or additional fields are set elsewhere.

#### Node: Flag for Manual Review
- **Type / role:** Slack node (notification).
- **Configuration:**
  - Posts to channel `#translation-review`
  - Message includes title, language pair, quality score, issues list.
- **Inputs / outputs:**
  - **Input:** Quality Approved? (false branch)
  - **Output →** Aggregate All Translations
- **Credential requirements:**
  - Slack credentials/token with permission to post to the channel (or Slack app).
- **Edge cases / failures:**
  - Channel not found, missing scopes (`chat:write`), workspace permission issues.

#### Node: Aggregate All Translations
- **Type / role:** Aggregate node (collects all items into one).
- **Configuration:**
  - Mode: **Aggregate all item data** (collects full items into an array; output commonly uses `data`).
- **Inputs / outputs:**
  - **Inputs:** Publish to WordPress, Flag for Manual Review
  - **Output →** Log to Translation Memory
- **Edge cases:**
  - If some branches error before reaching aggregation, counts/logs may be incomplete.
  - Large batches can create a large aggregated payload.

#### Node: Log to Translation Memory
- **Type / role:** Google Sheets append (logging / translation memory).
- **Configuration:**
  - Operation: **Append**
  - Document and Sheet are not selected in the JSON (left blank), so must be configured.
  - Intended headers (from sticky note): `contentId, title, sourceLanguage, targetLanguage, translatedText, qualityScore`
- **Inputs / outputs:**
  - **Input:** Aggregate All Translations
  - **Output →** Send Completion Summary
- **Credential requirements:**
  - Google Sheets OAuth2 credentials.
- **Edge cases / failures:**
  - Not configured doc/sheet => node will fail at runtime.
  - Aggregated structure: the node receives one item with `data: [...]`; you must map rows properly. As-is, it may append a single row with a JSON blob unless configured with field mapping.

#### Node: Send Completion Summary
- **Type / role:** Slack node (batch summary notification).
- **Configuration:**
  - Posts to `#content-updates`
  - Uses expressions on aggregated array:
    - Total: `{{ $json.data.length }}`
    - Approved: `{{ $json.data.filter(t => t.isApproved).length }}`
    - Needs Review: `{{ $json.data.filter(t => !t.isApproved).length }}`
- **Inputs / outputs:**
  - **Input:** Log to Translation Memory
  - **Output →** Respond to Webhook
- **Edge cases:**
  - If `data` is missing or not an array, expressions will throw.

#### Node: Respond to Webhook
- **Type / role:** RespondToWebhook (returns HTTP response to caller).
- **Configuration:**
  - Respond with: JSON
  - Body:
    - `success: true`
    - `translationsProcessed: $json.data.length`
    - `approved: $json.data.filter(t => t.isApproved).length`
- **Inputs / outputs:**
  - **Input:** Send Completion Summary
  - No downstream outputs.
- **Edge cases:**
  - For scheduled executions, this node may be irrelevant; for webhook executions it is required to avoid client timeouts.
  - Same `data` dependency as summary node.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Main Overview1 | Sticky Note | Documentation / overview | — | — | ## How it works… (translation pipeline, setup steps incl. credentials, config node, Sheets headers, Slack channels) |
| Section: Config1 | Sticky Note | Section label | — | — | ## 1. Setup & Logic Defines translation parameters and splits the workflow… |
| Section: Translation1 | Sticky Note | Section label | — | — | ## 2. Translation Connects to DeepL API… |
| Section: AI QA1 | Sticky Note | Section label | — | — | ## 3. AI Quality Guard AI Agent reviews… |
| Section: Distribution1 | Sticky Note | Section label | — | — | ## 4. Output & Logging Publishes approved content, flags issues… |
| Content Webhook | Webhook | HTTP trigger for inbound content | — | Merge Triggers | ## 1. Setup & Logic Defines translation parameters and splits the workflow… |
| Scheduled Batch | Schedule Trigger | Time-based trigger (every 4h) | — | Merge Triggers | ## 1. Setup & Logic Defines translation parameters and splits the workflow… |
| Merge Triggers | Merge | Unify webhook/schedule paths | Content Webhook, Scheduled Batch | Translation Configuration | ## 1. Setup & Logic Defines translation parameters and splits the workflow… |
| Translation Configuration | Set | Central config for languages/threshold | Merge Triggers | Prepare Source Content | ## 1. Setup & Logic Defines translation parameters and splits the workflow… |
| Prepare Source Content | Code | Normalize payload into internal schema | Translation Configuration | Split Target Languages | ## 1. Setup & Logic Defines translation parameters and splits the workflow… |
| Split Target Languages | SplitOut | Fan-out per target language | Prepare Source Content | DeepL Translate | ## 1. Setup & Logic Defines translation parameters and splits the workflow… |
| DeepL Translate | DeepL | Translate text into target language | Split Target Languages | Format Translation Result | ## 2. Translation Connects to DeepL API… |
| Format Translation Result | Code | Standardize DeepL output + metadata | DeepL Translate | AI Quality Reviewer | ## 2. Translation Connects to DeepL API… |
| OpenAI Chat Model | LangChain OpenAI Chat Model | Provides LLM to agent | — | AI Quality Reviewer (ai_languageModel) | ## 3. AI Quality Guard AI Agent reviews… |
| AI Quality Reviewer | LangChain Agent | QA scoring + approval decision | Format Translation Result, OpenAI Chat Model | Parse Quality Result | ## 3. AI Quality Guard AI Agent reviews… |
| Parse Quality Result | Code | Parse agent JSON; set isApproved | AI Quality Reviewer | Quality Approved? | ## 3. AI Quality Guard AI Agent reviews… |
| Quality Approved? | IF | Branch on approval boolean | Parse Quality Result | Publish to WordPress, Flag for Manual Review | ## 4. Output & Logging Publishes approved content, flags issues… |
| Publish to WordPress | WordPress | Create draft for approved translation | Quality Approved? | Aggregate All Translations | ## 4. Output & Logging Publishes approved content, flags issues… |
| Flag for Manual Review | Slack | Notify review channel for non-approved | Quality Approved? | Aggregate All Translations | ## 4. Output & Logging Publishes approved content, flags issues… |
| Aggregate All Translations | Aggregate | Combine all language results into one item | Publish to WordPress, Flag for Manual Review | Log to Translation Memory | ## 4. Output & Logging Publishes approved content, flags issues… |
| Log to Translation Memory | Google Sheets | Append translation results to sheet | Aggregate All Translations | Send Completion Summary | ## 4. Output & Logging Publishes approved content, flags issues… |
| Send Completion Summary | Slack | Post batch summary | Log to Translation Memory | Respond to Webhook | ## 4. Output & Logging Publishes approved content, flags issues… |
| Respond to Webhook | Respond to Webhook | Return JSON response to caller | Send Completion Summary | — | ## 4. Output & Logging Publishes approved content, flags issues… |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it: **Multi-Language Content Translation Pipeline with AI Quality Control**
   - Keep it inactive until credentials and nodes are configured.

2. **Add triggers**
   1) Add **Webhook** node named **Content Webhook**
   - Method: **POST**
   - Path: `translate-content`
   - Response: **Using “Respond to Webhook” node**
   - (Optional) On Error: continue (to match behavior)

   2) Add **Schedule Trigger** node named **Scheduled Batch**
   - Interval: every **4 hours**

3. **Merge trigger paths**
   - Add **Merge** node named **Merge Triggers**
   - Mode: **Choose Branch**
   - Connect:
     - Content Webhook → Merge Triggers (Input 1)
     - Scheduled Batch → Merge Triggers (Input 2)

4. **Add configuration**
   - Add **Set** node named **Translation Configuration**
   - Mode: “Raw JSON”
   - Paste config values:
     - sourceLanguage = EN
     - targetLanguages = ["DE","FR","ES","JA"]
     - contentType = "article"
     - preserveFormatting = true
     - qualityThreshold = 0.85
     - distributionTargets = ["wordpress","contentful"]
   - Connect Merge Triggers → Translation Configuration

5. **Normalize incoming content**
   - Add **Code** node named **Prepare Source Content**
   - Implement logic to:
     - Read incoming item fields `content` or `text`, fallback text
     - Set `title`, `contentId`, timestamps
     - Attach config from Translation Configuration
   - Connect Translation Configuration → Prepare Source Content

6. **Fan-out by language**
   - Add **Split Out** node named **Split Target Languages**
   - Field to split: `targetLanguages`
   - Connect Prepare Source Content → Split Target Languages

7. **Translate with DeepL**
   - Add **DeepL** node named **DeepL Translate**
   - Configure credentials: **DeepL API**
   - Text: expression referencing prepared source text (from Prepare Source Content)
   - Target language: expression set to the split item value (the language code)
   - Connect Split Target Languages → DeepL Translate

8. **Format translation output**
   - Add **Code** node named **Format Translation Result**
   - Extract translated text from DeepL response and output a normalized object:
     - contentId, title, sourceLanguage, targetLanguage, sourceText, translatedText, etc.
   - Connect DeepL Translate → Format Translation Result

9. **Set up OpenAI model for LangChain**
   - Add **OpenAI Chat Model** node (LangChain) named **OpenAI Chat Model**
   - Credentials: OpenAI API key
   - Model: `gpt-4o-mini`
   - Temperature: `0.2`

10. **AI QA agent**
   - Add **AI Agent** node (LangChain) named **AI Quality Reviewer**
   - Set the evaluation prompt that requests *strict JSON output* including:
     - qualityScore, accuracyScore, fluencyScore, issues, suggestions, approved, summary
   - Add system message: professional translator/QA persona
   - Connect:
     - Format Translation Result → AI Quality Reviewer (main)
     - OpenAI Chat Model → AI Quality Reviewer (ai_languageModel port)

11. **Parse QA result**
   - Add **Code** node named **Parse Quality Result**
   - Implement:
     - Extract JSON from the agent output (regex + JSON.parse)
     - Fallback behavior (decide whether you want “default approve” or “default reject”)
     - Output `quality` object + `isApproved`
   - Connect AI Quality Reviewer → Parse Quality Result

12. **Branch on approval**
   - Add **IF** node named **Quality Approved?**
   - Condition: boolean `isApproved` equals `true`
   - Connect Parse Quality Result → Quality Approved?

13. **Approved path: WordPress**
   - Add **WordPress** node named **Publish to WordPress**
   - Credentials: WordPress (site URL + auth method supported by your n8n node)
   - Create post as **draft**
   - Title: `{{title}} ({{targetLanguage}})`
   - (Recommended) Map **translatedText** into the post content/body so drafts are not empty.
   - Connect Quality Approved? (true) → Publish to WordPress

14. **Rejected path: Slack review**
   - Add **Slack** node named **Flag for Manual Review**
   - Credentials: Slack bot/app with `chat:write`
   - Post message to channel `#translation-review`
   - Include title, language pair, quality score, issues.
   - Connect Quality Approved? (false) → Flag for Manual Review

15. **Aggregate all results**
   - Add **Aggregate** node named **Aggregate All Translations**
   - Operation/mode: **Aggregate all item data**
   - Connect:
     - Publish to WordPress → Aggregate All Translations
     - Flag for Manual Review → Aggregate All Translations

16. **Log to Google Sheets (translation memory)**
   - Add **Google Sheets** node named **Log to Translation Memory**
   - Credentials: Google OAuth2
   - Operation: **Append**
   - Select Spreadsheet (Document) and Worksheet (Sheet)
   - Ensure the sheet has headers (per workflow note):
     - `contentId, title, sourceLanguage, targetLanguage, translatedText, qualityScore`
   - Map fields appropriately; because aggregation produces `data: [ ... ]`, you may need to:
     - either split `data` back into items before appending, or
     - configure appending in a way that writes multiple rows from the array (depending on node capability/version).
   - Connect Aggregate All Translations → Log to Translation Memory

17. **Send Slack completion summary**
   - Add **Slack** node named **Send Completion Summary**
   - Channel: `#content-updates`
   - Use expressions referencing `$json.data` to compute totals.
   - Connect Log to Translation Memory → Send Completion Summary

18. **Respond to webhook**
   - Add **Respond to Webhook** node named **Respond to Webhook**
   - Respond with JSON indicating success + counts derived from `$json.data`.
   - Connect Send Completion Summary → Respond to Webhook

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| This workflow automates a professional-grade translation pipeline: trigger (webhook/schedule) → DeepL translation → AI QA → WordPress draft or Slack flag → Google Sheets logging → Slack summary. | Sticky note “How it works” |
| Setup: connect credentials for DeepL, OpenAI, WordPress, Slack, Google Sheets. | Sticky note “Setup steps” |
| Configure languages and parameters in “Translation Configuration” (e.g., DE, FR, JA). | Sticky note “Setup steps” |
| Google Sheets should include headers: contentId, title, sourceLanguage, targetLanguage, translatedText, qualityScore. | Sticky note “Setup steps” |
| Slack: select notification channels in “Flag for Manual Review” and “Send Completion Summary”. | Sticky note “Setup steps” |