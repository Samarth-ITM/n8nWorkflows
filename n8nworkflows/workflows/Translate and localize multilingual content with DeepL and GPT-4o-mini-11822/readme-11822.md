Translate and localize multilingual content with DeepL and GPT-4o-mini

https://n8nworkflows.xyz/workflows/translate-and-localize-multilingual-content-with-deepl-and-gpt-4o-mini-11822


# Translate and localize multilingual content with DeepL and GPT-4o-mini

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:** This workflow receives a translation request via HTTP, validates target languages, translates the source text into multiple languages in parallel using **DeepL**, runs an **AI quality review** with **GPT-4o-mini**, applies basic localization metadata, aggregates all results, optionally logs a summary to **Google Sheets**, sends a completion email via **Gmail**, and returns a unified JSON response to the caller.

**Typical use cases**
- Marketing/content teams translating copy into multiple locales
- Agencies producing multilingual variants with a lightweight automated QA score
- Product teams localizing help text or announcements with centralized reporting

### Logical Blocks
1. **1.1 Content Intake & Job Initialization**
   - Webhook entry, normalize input into a job object, validate/clean language codes, detect a rough source language.
2. **1.2 Parallel Translation (per target language)**
   - Split target languages into one item per language, call DeepL, normalize DeepL response.
3. **1.3 AI Quality Review (per translation)**
   - GPT-4o-mini reviews translation and returns JSON with score/issues/suggestions; parsing and “needs human review” flag.
4. **1.4 Localization, Aggregation, Logging, Notification & Response**
   - Apply locale settings (metadata), aggregate per-language results, build a summary, log to Sheets, email notification, respond to the webhook.

---

## 2. Block-by-Block Analysis

### 1.1 Content Intake & Job Initialization

**Overview:** Receives the POST request, creates a job envelope (jobId + normalized fields), validates target languages, and performs a simple heuristic source-language detection.

**Nodes Involved:**
- Sticky Note
- Sticky Note1
- Translation Request
- Initialize Translation Job
- Detect Source Language

#### Node: Sticky Note
- **Type / Role:** Sticky Note (documentation)
- **Configuration choices:** Describes audience, steps, setup requirements.
- **Connections:** None
- **Failure modes:** None

#### Node: Sticky Note1
- **Type / Role:** Sticky Note (documentation for Step 1)
- **Connections:** None

#### Node: Translation Request
- **Type / Role:** `Webhook` (workflow entry point)
- **Key config:**
  - **Method:** POST
  - **Path:** `/translate`
  - **Response mode:** “Respond with Respond to Webhook node” (responseNode)
- **Inputs/Outputs:**
  - **Output →** Initialize Translation Job
- **Expected request body (implicit from expressions):**
  - `body.sourceText` (string)
  - `body.targetLanguages` (array of language codes)
  - `body.contentType` (optional string)
  - `body.glossaryId` (optional string)
- **Failure/edge cases:**
  - Missing `body` or fields leads to empty `sourceText` and potentially validation failure later.
  - If caller expects immediate response, note workflow does multiple downstream calls (DeepL + OpenAI), which can increase webhook latency/timeouts.

#### Node: Initialize Translation Job
- **Type / Role:** `Set` (normalize input into a consistent schema)
- **Key config choices:**
  - `jobId`: `TRN-` + timestamp via `$now.format('yyyyMMddHHmmss')`
  - `sourceText`: from `$json.body.sourceText`
  - `targetLanguages`: from `$json.body.targetLanguages` (kept as “object” type in node UI; effectively an array)
  - `contentType`: defaults to `'general'`
  - `glossaryId`: defaults to `''`
  - `createdAt`: `$now.toISO()`
- **Output →** Detect Source Language
- **Failure/edge cases:**
  - If `targetLanguages` is not an array, downstream filtering/splitting may behave unexpectedly.

#### Node: Detect Source Language
- **Type / Role:** `Code` (validate targets + heuristic source detection)
- **Logic (interpreted):**
  - Reads `targetLanguages`; defaults to `['en']` if missing.
  - Validates against allowlist: `en, ja, de, fr, es, zh, ko, pt, it, nl, pl, ru`.
  - Normalizes by `toLowerCase()` and filters only valid.
  - **Throws error** if no valid targets remain.
  - Heuristic detection:
    - If Japanese/Chinese characters detected → `ja`
    - Else if Cyrillic detected → `ru`
    - Else default → `en`
  - Outputs:
    - `detectedSourceLang`
    - `validatedTargetLangs` (array)
    - `languageCount`
- **Output →** Split by Target Language
- **Failure/edge cases:**
  - Throws: “No valid target languages specified”.
  - Heuristic detection is simplistic; Chinese text may be labeled `ja` due to the combined regex range.
  - If `sourceText` is empty, detected language defaults to `en` and translation may be poor/useless.

---

### 1.2 Parallel Translation (per target language)

**Overview:** Converts the validated target language array into one item per language, calls DeepL for each, and maps the translation response into a consistent structure for later scoring.

**Nodes Involved:**
- Sticky Note2
- Split by Target Language
- Prepare Translation Request
- DeepL Translate
- Process Translation Result

#### Node: Sticky Note2
- **Type / Role:** Sticky Note (documentation for Step 2)
- **Connections:** None

#### Node: Split by Target Language
- **Type / Role:** `Split Out` (fan-out into per-language items)
- **Key config:**
  - `fieldToSplitOut`: `validatedTargetLangs`
  - `include`: “allOtherFields” (keeps job context fields alongside each split item)
- **Input:** From Detect Source Language
- **Output →** Prepare Translation Request
- **Failure/edge cases:**
  - If `validatedTargetLangs` is not an array, split may fail or produce unexpected items.

#### Node: Prepare Translation Request
- **Type / Role:** `Set` (set current target language for this item)
- **Key config:**
  - Adds `currentTargetLang` = `{{$json.validatedTargetLangs}}`  
    (At this point `validatedTargetLangs` is the *single split value* per item.)
  - Includes other fields (jobId, sourceText, etc.)
- **Output →** DeepL Translate
- **Failure/edge cases:**
  - Naming is a bit confusing (field still called `validatedTargetLangs` but holds a single value after split). The node mitigates this by creating `currentTargetLang`.

#### Node: DeepL Translate
- **Type / Role:** `DeepL` (machine translation API call)
- **Key config:**
  - `text`: `{{$json.sourceText}}`
  - `translateTo`: `{{$json.currentTargetLang}}`
  - No additional fields configured (not using glossary/formality, etc.)
- **Input:** Prepare Translation Request
- **Output →** Process Translation Result
- **Credentials required:** DeepL API credentials in n8n
- **Failure/edge cases:**
  - Auth errors (missing/invalid DeepL key)
  - Rate limiting / quota exceeded
  - Target language code mismatches with DeepL accepted codes (workflow validates against its own allowlist, which may not perfectly match DeepL’s expected variants, e.g., `pt` vs `pt-PT/pt-BR` depending on account/settings).
  - Large text length constraints.

#### Node: Process Translation Result
- **Type / Role:** `Code` (normalize DeepL output + merge context)
- **Key expressions/variables:**
  - Pulls previous context via `$('Prepare Translation Request').first().json`
- **Output fields:**
  - `jobId`, `sourceText`, `targetLang` (`currentTargetLang`)
  - `translatedText`: from `input.translations[0].text` or fallback `input.text` or `''`
  - `detectedSourceLang`: from DeepL `detected_source_language` or previous heuristic
  - `contentType`, `glossaryId`
- **Output →** AI Quality Review
- **Failure/edge cases:**
  - If DeepL returns a different schema, `translatedText` may become empty.
  - Using `.first()` from “Prepare Translation Request” assumes one-to-one pairing; with parallelization it typically works per item, but if node execution context changes (or multiple items merge), this pattern can be fragile.

---

### 1.3 AI Quality Review (per translation)

**Overview:** Uses an AI Agent backed by GPT-4o-mini to score translation quality and produce structured feedback; then parses the model output robustly and flags low-score items.

**Nodes Involved:**
- Sticky Note3
- OpenAI Chat Model
- AI Quality Review
- Parse Quality Score

#### Node: Sticky Note3
- **Type / Role:** Sticky Note (documentation for Step 3)
- **Connections:** None

#### Node: OpenAI Chat Model
- **Type / Role:** LangChain Chat Model (`lmChatOpenAi`) used as the AI Agent’s language model
- **Key config:**
  - `model`: `gpt-4o-mini`
  - `temperature`: `0.3` (more consistent scoring/JSON formatting)
- **Connections:**
  - Provides **ai_languageModel** input to **AI Quality Review**
- **Credentials required:** OpenAI API credentials in n8n
- **Failure/edge cases:**
  - Missing/invalid OpenAI credentials
  - Model availability changes or org restrictions
  - Rate limiting/timeouts

#### Node: AI Quality Review
- **Type / Role:** LangChain `agent` (runs the review prompt and expects JSON-only output)
- **Key config:**
  - Prompt includes source language, source text, target language, translated text, content type
  - Asks for 1–10 scores and returns JSON: `{ score, issues, suggestions }`
  - **System message:** “You are a professional translator… Output valid JSON only.”
  - `promptType`: define
- **Inputs/Outputs:**
  - **Main input:** from Process Translation Result
  - **ai_languageModel:** from OpenAI Chat Model
  - **Main output →** Parse Quality Score
- **Failure/edge cases:**
  - Model may still output non-JSON; downstream parser attempts recovery.
  - Large texts may increase cost/latency and risk truncation.

#### Node: Parse Quality Score
- **Type / Role:** `Code` (extract JSON from AI output and merge into translation record)
- **Logic:**
  - Reads `input.output` (agent output) default `{}` string
  - Extracts first `{ ... }` block via regex and `JSON.parse`
  - Fallback defaults: `{ score: 7, issues: [], suggestions: [] }`
  - Sets:
    - `qualityScore`
    - `issues`, `suggestions`
    - `needsReview`: score < 6
  - Merges with original translation data from `$('Process Translation Result').first().json`
- **Output →** Apply Localization
- **Failure/edge cases:**
  - If agent output doesn’t include braces, defaults apply (may hide failures).
  - `.first()` cross-node reference has the same potential fragility as above.

---

### 1.4 Localization, Aggregation, Logging, Notification & Response

**Overview:** Adds locale metadata, aggregates per-language items, builds a single summary object, logs summary to Google Sheets, emails the team, and returns JSON to the webhook caller.

**Nodes Involved:**
- Sticky Note4
- Apply Localization
- Aggregate All Translations
- Build Translation Summary
- Log to Google Sheets
- Send Completion Email
- Return Results

#### Node: Sticky Note4
- **Type / Role:** Sticky Note (documentation for Step 4)
- **Connections:** None

#### Node: Apply Localization
- **Type / Role:** `Code` (apply simple localization settings)
- **Logic:**
  - Defines `localizationRules` for: `ja, de, fr, es, zh` with `dateFormat` and `currency`
  - Sets:
    - `localizationApplied` (true if rules exist)
    - `localizedText`: currently equals `translatedText` (no actual text transformation)
    - `localeSettings`: rules object (or `{}`)
    - `processedAt`: timestamp
- **Output →** Aggregate All Translations
- **Failure/edge cases:**
  - This is metadata-only; if true localization is required (dates/numbers), additional parsing/formatting nodes are needed.

#### Node: Aggregate All Translations
- **Type / Role:** `Aggregate` (collect all per-language items back into one)
- **Key config:**
  - Aggregation mode: “aggregateAllItemData”
  - Destination field: `translations` (array of all items)
- **Output →** Build Translation Summary
- **Failure/edge cases:**
  - If no items reach aggregation (e.g., upstream errors), summary building can divide by zero later.

#### Node: Build Translation Summary
- **Type / Role:** `Code` (create final job summary + results array)
- **Output summary fields:**
  - `jobId`, `sourceText`
  - `totalLanguages`
  - `averageQualityScore`: sum(scores)/count
  - `needsReviewCount`
  - `results`: array of `{ language, text, score, needsReview }`
  - `completedAt`
- **Outputs →**
  - Log to Google Sheets
  - Send Completion Email
- **Failure/edge cases:**
  - If `translations.length === 0`, `averageQualityScore` becomes `NaN` (division by zero).
  - Truncation assumptions: uses first translation to get `jobId/sourceText`.

#### Node: Log to Google Sheets
- **Type / Role:** `Google Sheets` (append summary row)
- **Key config:**
  - Operation: Append
  - Sheet: “Translations”
  - Mapped columns:
    - Job ID, Completed, Languages, Avg Quality (toFixed(1)), Source Text (first 200 chars)
  - `documentId` is empty in the workflow export (must be set)
- **Output →** Return Results
- **Credentials required:** Google Sheets OAuth2/service account depending on n8n setup
- **Failure/edge cases:**
  - Missing `documentId` or wrong sheet name causes runtime error.
  - Auth/permission errors.
  - `averageQualityScore` being `NaN` will break `toFixed(1)`.

#### Node: Send Completion Email
- **Type / Role:** `Gmail` (notification)
- **Key config:**
  - To: `team@company.com` (static)
  - Subject: `Translation Complete: {{ $json.jobId }}`
  - Body includes job stats and per-language scores.
- **Input:** Build Translation Summary
- **Credentials required:** Gmail OAuth2 in n8n
- **Failure/edge cases:**
  - Auth errors / restricted sending identities
  - If results array is missing, mapping expression may fail

#### Node: Return Results
- **Type / Role:** `Respond to Webhook` (final HTTP response)
- **Key config:**
  - Respond with: JSON
  - Response body expression builds:
    - `success`, `jobId`, `totalLanguages`, `averageQuality`, `results`
  - The expression wraps with `JSON.stringify(...)` even though “respondWith: json” is selected; this may cause the caller to receive a JSON *string* rather than an object depending on n8n behavior/version.
- **Input:** Log to Google Sheets
- **Failure/edge cases:**
  - If Google Sheets fails, the workflow will not reach this node (no error handling branches present), leaving webhook caller without a proper response.
  - Double-encoding risk due to `JSON.stringify` in a JSON response mode.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note | n8n-nodes-base.stickyNote | Documentation / overview |  |  | ## Translation & Localization with DeepL and GPT-4o-mini; Who is this for?; What this workflow does; Setup; Requirements |
| Sticky Note1 | n8n-nodes-base.stickyNote | Documentation (Step 1) |  |  | Step 1: Content Intake – Receive translation request via webhook and validate target language codes. |
| Sticky Note2 | n8n-nodes-base.stickyNote | Documentation (Step 2) |  |  | Step 2: Parallel Translation – Split Out creates parallel streams for each language. DeepL translates with glossary support. |
| Sticky Note3 | n8n-nodes-base.stickyNote | Documentation (Step 3) |  |  | Step 3: Quality Review – AI Agent evaluates accuracy, fluency, and style. Flags items needing human review. |
| Sticky Note4 | n8n-nodes-base.stickyNote | Documentation (Step 4) |  |  | Step 4: Localization & Output – Apply cultural adaptations, aggregate results, log to Sheets, and send notification. |
| Translation Request | n8n-nodes-base.webhook | HTTP entry point for translation job |  | Initialize Translation Job | Step 1: Content Intake – Receive translation request via webhook and validate target language codes. |
| Initialize Translation Job | n8n-nodes-base.set | Normalize request into job fields | Translation Request | Detect Source Language | Step 1: Content Intake – Receive translation request via webhook and validate target language codes. |
| Detect Source Language | n8n-nodes-base.code | Validate target languages + heuristic source detection | Initialize Translation Job | Split by Target Language | Step 1: Content Intake – Receive translation request via webhook and validate target language codes. |
| Split by Target Language | n8n-nodes-base.splitOut | Fan-out per target language | Detect Source Language | Prepare Translation Request | Step 2: Parallel Translation – Split Out creates parallel streams for each language. DeepL translates with glossary support. |
| Prepare Translation Request | n8n-nodes-base.set | Set `currentTargetLang` for this item | Split by Target Language | DeepL Translate | Step 2: Parallel Translation – Split Out creates parallel streams for each language. DeepL translates with glossary support. |
| DeepL Translate | n8n-nodes-base.deepL | Translate text via DeepL API | Prepare Translation Request | Process Translation Result | Step 2: Parallel Translation – Split Out creates parallel streams for each language. DeepL translates with glossary support. |
| Process Translation Result | n8n-nodes-base.code | Map DeepL response into normalized translation object | DeepL Translate | AI Quality Review | Step 2: Parallel Translation – Split Out creates parallel streams for each language. DeepL translates with glossary support. |
| OpenAI Chat Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | Provides GPT-4o-mini model to agent |  | AI Quality Review (ai_languageModel) | Step 3: Quality Review – AI Agent evaluates accuracy, fluency, and style. Flags items needing human review. |
| AI Quality Review | @n8n/n8n-nodes-langchain.agent | AI-based translation evaluation returning JSON | Process Translation Result; OpenAI Chat Model (ai) | Parse Quality Score | Step 3: Quality Review – AI Agent evaluates accuracy, fluency, and style. Flags items needing human review. |
| Parse Quality Score | n8n-nodes-base.code | Parse/repair AI JSON + flag `needsReview` | AI Quality Review | Apply Localization | Step 3: Quality Review – AI Agent evaluates accuracy, fluency, and style. Flags items needing human review. |
| Apply Localization | n8n-nodes-base.code | Attach locale settings + produce `localizedText` | Parse Quality Score | Aggregate All Translations | Step 4: Localization & Output – Apply cultural adaptations, aggregate results, log to Sheets, and send notification. |
| Aggregate All Translations | n8n-nodes-base.aggregate | Gather all per-language items into one array | Apply Localization | Build Translation Summary | Step 4: Localization & Output – Apply cultural adaptations, aggregate results, log to Sheets, and send notification. |
| Build Translation Summary | n8n-nodes-base.code | Compute averages and final response structure | Aggregate All Translations | Log to Google Sheets; Send Completion Email | Step 4: Localization & Output – Apply cultural adaptations, aggregate results, log to Sheets, and send notification. |
| Log to Google Sheets | n8n-nodes-base.googleSheets | Append summary row to spreadsheet | Build Translation Summary | Return Results | Step 4: Localization & Output – Apply cultural adaptations, aggregate results, log to Sheets, and send notification. |
| Send Completion Email | n8n-nodes-base.gmail | Email notification to team | Build Translation Summary |  | Step 4: Localization & Output – Apply cultural adaptations, aggregate results, log to Sheets, and send notification. |
| Return Results | n8n-nodes-base.respondToWebhook | HTTP response to caller | Log to Google Sheets |  | Step 4: Localization & Output – Apply cultural adaptations, aggregate results, log to Sheets, and send notification. |

---

## 4. Reproducing the Workflow from Scratch

1. **Create Workflow**
   - Name: **“Translation & Localization with DeepL and GPT-4o-mini”**
   - (Optional) Add Sticky Notes to label steps and requirements.

2. **Add Webhook node**
   - Node type: **Webhook**
   - Name: **Translation Request**
   - Method: **POST**
   - Path: **translate**
   - Response mode: **Using “Respond to Webhook” node**

3. **Add Set node (job initialization)**
   - Node type: **Set**
   - Name: **Initialize Translation Job**
   - Add fields:
     - `jobId` (string): `{{ 'TRN-' + $now.format('yyyyMMddHHmmss') }}`
     - `sourceText` (string): `{{ $json.body.sourceText }}`
     - `targetLanguages` (object/array): `{{ $json.body.targetLanguages }}`
     - `contentType` (string): `{{ $json.body.contentType || 'general' }}`
     - `glossaryId` (string): `{{ $json.body.glossaryId || '' }}`
     - `createdAt` (string): `{{ $now.toISO() }}`
   - Connect: **Translation Request → Initialize Translation Job**

4. **Add Code node (validate languages + detect source)**
   - Node type: **Code**
   - Name: **Detect Source Language**
   - Paste logic that:
     - filters target languages against an allowlist
     - throws if none are valid
     - sets `detectedSourceLang`, `validatedTargetLangs`, `languageCount`
   - Connect: **Initialize Translation Job → Detect Source Language**

5. **Add Split Out node (fan-out)**
   - Node type: **Split Out**
   - Name: **Split by Target Language**
   - Field to split out: `validatedTargetLangs`
   - Include: **All other fields**
   - Connect: **Detect Source Language → Split by Target Language**

6. **Add Set node (per-item target language)**
   - Node type: **Set**
   - Name: **Prepare Translation Request**
   - Keep other fields: enabled
   - Add:
     - `currentTargetLang` (string): `{{ $json.validatedTargetLangs }}`
   - Connect: **Split by Target Language → Prepare Translation Request**

7. **Add DeepL node (translation)**
   - Node type: **DeepL**
   - Name: **DeepL Translate**
   - Credentials: configure **DeepL API** in n8n credentials
   - Text: `{{ $json.sourceText }}`
   - Translate to: `{{ $json.currentTargetLang }}`
   - Connect: **Prepare Translation Request → DeepL Translate**

8. **Add Code node (normalize translation output)**
   - Node type: **Code**
   - Name: **Process Translation Result**
   - Map DeepL response to:
     - `jobId`, `sourceText`, `targetLang`, `translatedText`, `detectedSourceLang`, `contentType`, `glossaryId`
   - Connect: **DeepL Translate → Process Translation Result**

9. **Add OpenAI Chat Model node**
   - Node type: **OpenAI Chat Model (LangChain)**
   - Name: **OpenAI Chat Model**
   - Credentials: configure **OpenAI API** in n8n credentials
   - Model: **gpt-4o-mini**
   - Temperature: **0.3**

10. **Add AI Agent node (quality review)**
   - Node type: **AI Agent (LangChain)**
   - Name: **AI Quality Review**
   - System message: “You are a professional translator… Output valid JSON only.”
   - Prompt: include source/translation and request JSON `{ score, issues, suggestions }`
   - Connect:
     - **Process Translation Result → AI Quality Review (main)**
     - **OpenAI Chat Model → AI Quality Review (ai_languageModel)**

11. **Add Code node (parse AI output)**
   - Node type: **Code**
   - Name: **Parse Quality Score**
   - Parse `input.output`, recover JSON with regex, default score if parsing fails, set `needsReview` when score < 6.
   - Connect: **AI Quality Review → Parse Quality Score**

12. **Add Code node (localization metadata)**
   - Node type: **Code**
   - Name: **Apply Localization**
   - Add locale rules and output:
     - `localizedText` (currently same as translatedText)
     - `localeSettings`, `localizationApplied`, `processedAt`
   - Connect: **Parse Quality Score → Apply Localization**

13. **Add Aggregate node**
   - Node type: **Aggregate**
   - Name: **Aggregate All Translations**
   - Mode: **Aggregate all item data**
   - Destination field: **translations**
   - Connect: **Apply Localization → Aggregate All Translations**

14. **Add Code node (build summary)**
   - Node type: **Code**
   - Name: **Build Translation Summary**
   - Compute:
     - `totalLanguages`, `averageQualityScore`, `needsReviewCount`, `results[]`, `completedAt`
   - Connect: **Aggregate All Translations → Build Translation Summary**

15. **Add Google Sheets node (optional but wired in this workflow)**
   - Node type: **Google Sheets**
   - Name: **Log to Google Sheets**
   - Credentials: configure Google Sheets OAuth2/service account
   - Operation: **Append**
   - Document: select your spreadsheet (**documentId must be set**)
   - Sheet: **Translations**
   - Map columns:
     - Job ID, Completed, Languages, Avg Quality, Source Text (truncated)
   - Connect: **Build Translation Summary → Log to Google Sheets**

16. **Add Gmail node (notification)**
   - Node type: **Gmail**
   - Name: **Send Completion Email**
   - Credentials: configure Gmail OAuth2
   - To: `team@company.com`
   - Subject/body: use expressions referencing summary fields.
   - Connect: **Build Translation Summary → Send Completion Email**

17. **Add Respond to Webhook node**
   - Node type: **Respond to Webhook**
   - Name: **Return Results**
   - Respond with: **JSON**
   - Body: build `{ success, jobId, totalLanguages, averageQuality, results }`
   - Connect: **Log to Google Sheets → Return Results**

**Important implementation note:** If you want the response to be a JSON object (not a JSON string), set the response body to the object directly (avoid `JSON.stringify(...)`) when using “Respond with: JSON”.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Requires DeepL API credentials, OpenAI API credentials (GPT-4o-mini), Google Sheets (optional), Gmail | From workflow sticky note “Setup” and “Requirements” |
| Language validation allowlist is hard-coded; source language detection is heuristic | Detect Source Language code logic |
| No explicit error-handling branches; failures in DeepL/OpenAI/Sheets/Gmail can prevent webhook response | Overall design consideration |
| Google Sheets node has empty document selection in the provided workflow; must be set before running | Log to Google Sheets configuration |
| RespondToWebhook node uses JSON mode but body is `JSON.stringify(...)` which can double-encode | Return Results configuration |