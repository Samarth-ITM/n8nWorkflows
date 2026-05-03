Generate illustrated stories with GPT-4, DALL-E 3 and Firebase

https://n8nworkflows.xyz/workflows/generate-illustrated-stories-with-gpt-4--dall-e-3-and-firebase-12263


# Generate illustrated stories with GPT-4, DALL-E 3 and Firebase

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:**  
This workflow generates a full illustrated story from user inputs (topic, language, audience, mood, style, number of scenes). It uses **OpenAI GPT-4o** to produce a structured story (JSON), **DALL·E 3** to generate one image per scene, uploads images to **Firebase Storage**, stores the complete story in **Firestore**, and returns a final JSON response to the user.

**Target use cases:**
- Story generation apps (kids, YA, adults) with per-scene illustrations
- Content pipelines that need structured story output + hosted image URLs
- Prototyping AI narrative + image generation workflows

### Logical blocks
1.1 **Input Reception (Form Trigger)**  
Collect user parameters via n8n Form.

1.2 **Validation & Story ID Creation**  
Clamp scene count to 1–12 and generate a unique `storyId`.

1.3 **Story Generation (GPT-4o)**  
Generate story content and prompts as strict JSON using OpenAI Chat Completions with `json_schema`.

1.4 **Prompt Preparation & Sanitization**  
Sanitize text and build DALL·E-ready prompts per scene.

1.5 **Image Generation (DALL·E 3)**  
Generate scene images sequentially, handle errors, and rate-limit.

1.6 **Firebase Storage Upload**  
Download each DALL·E image and upload it to Firebase Storage; replace scene image URLs with Firebase-hosted URLs.

1.7 **Finalize Story Object**  
Remove internal fields (prompt, indexes, errors) and build final story shape.

1.8 **Firestore Persistence**  
Convert story object to Firestore REST format and PATCH it into a document.

1.9 **Webhook Response**  
Return final JSON including the full story object.

---

## 2. Block-by-Block Analysis

### 2.1 Input Reception (Form Trigger)
**Overview:** Collects story requirements from the user and starts the workflow via a hosted form endpoint.

**Nodes involved:**
- Sticky Note1
- **Story Input Form**

#### Story Input Form
- **Type / role:** `n8n-nodes-base.formTrigger` — entrypoint; provides a user-facing form and triggers the workflow.
- **Key configuration choices:**
  - Form title: **“AI Story Generator”**
  - Description: explains 2–5 minutes generation time
  - On submit response text: “Your story is being created! This takes 2-5 minutes. Please wait...”
  - Fields:
    - Story Topic (textarea, required)
    - Language (dropdown, 12 options, required)
    - Image Style (dropdown, 10 options, required)
    - Number of Scenes (number, required)
    - Target Audience (dropdown, required)
    - Story Mood (dropdown, required)
- **Inputs/outputs:**
  - **No inputs** (trigger node)
  - Output → **Validate Input**
- **Edge cases / failures:**
  - User provides non-numeric scenes; later code defaults to 4.
  - Very long topic text may increase OpenAI latency/cost.
- **Version notes:** Node version `2.3` (formTrigger). Behavior depends on n8n form feature availability.

---

### 2.2 Validate & Story ID Creation
**Overview:** Normalizes and constrains `numScenes` and generates a unique `storyId` used across storage and Firestore.

**Nodes involved:**
- Sticky Note2
- **Validate Input**

#### Validate Input
- **Type / role:** `n8n-nodes-base.code` — input validation and enrichment.
- **Configuration choices (interpreted):**
  - Reads the first incoming item: `const formData = $input.first().json;`
  - Parses `Number of Scenes`; clamps to **1..12**, default **4**
  - Generates `storyId` using timestamp + random suffix:
    - `story_${Date.now()}_${Math.random().toString(36).substr(2, 9)}`
  - Returns merged object containing original form fields plus:
    - `numScenes`
    - `storyId`
- **Key variables/expressions:**
  - `$input.first().json`
- **Inputs/outputs:**
  - Input ← Story Input Form
  - Output → Generate Story (GPT-4)
- **Edge cases / failures:**
  - If the form field keys differ (renamed labels), e.g. `"Number of Scenes"` not found, parsing fails and defaults to 4.
  - This ID is not cryptographically unique; collisions are unlikely but possible at very high throughput.

---

### 2.3 Story Generation (GPT-4o)
**Overview:** Calls OpenAI Chat Completions to produce a strict JSON story object including scenes and image prompts.

**Nodes involved:**
- Sticky Note3
- **Generate Story (GPT-4)**

#### Generate Story (GPT-4)
- **Type / role:** `n8n-nodes-base.httpRequest` — OpenAI Chat Completions request.
- **Configuration choices:**
  - Method: `POST`
  - URL: `https://api.openai.com/v1/chat/completions`
  - Authentication: **predefined credential** `openAiApi` (n8n OpenAI credential)
  - Body contains:
    - `model: "gpt-4o"`
    - System message: storyteller instruction; emphasize DALL·E-friendly prompts
    - User message: injects form values via n8n expressions
    - Uses `response_format.type = "json_schema"` with `strict: true` to enforce structure:
      - Story metadata: title, summaries, language, audience, mood, moral
      - `scenes[]`: sceneNumber, title, text, textEnglish, imagePrompt
      - `characters[]`: name, description, role
- **Key expressions/variables:**
  - `{{ $json['Story Topic'] }}`
  - `{{ $json.Language }}`
  - `{{ $json.numScenes }}`
  - `{{ $json['Target Audience'] }}`
  - `{{ $json['Story Mood'] }}`
  - `{{ $json['Image Style'] }}`
- **Inputs/outputs:**
  - Input ← Validate Input
  - Output → Prepare Story Data
- **Edge cases / failures:**
  - OpenAI credential misconfiguration → 401/403.
  - Model or response_format incompatibility (depending on OpenAI API changes) → 400.
  - If OpenAI returns non-JSON content despite schema (rare but possible) downstream `JSON.parse` will fail.
  - Large `numScenes` (up to 12) increases token usage.
- **Version notes:** Node version `4.3` (HTTP Request). Uses n8n’s OpenAI credential type, not the dedicated OpenAI node.

---

### 2.4 Prompt Preparation & Sanitization
**Overview:** Parses the GPT output, sanitizes prompt text, and generates `dallePrompt` per scene while preserving metadata.

**Nodes involved:**
- Sticky Note4
- **Prepare Story Data**

#### Prepare Story Data
- **Type / role:** `n8n-nodes-base.code` — transforms GPT response into an internal structure for image generation.
- **Configuration choices:**
  - Reads OpenAI response from `$input.first().json`
  - Reads original form data via node reference:
    - `const formData = $('Validate Input').first().json;`
  - Extracts story JSON from:
    - `JSON.parse(response.choices[0].message.content)`
  - Sanitization function:
    - escapes backslashes, replaces double quotes with single quotes, removes control chars, normalizes whitespace
  - For each scene:
    - Adds `sceneIndex`
    - Builds `dallePrompt` combining selected image style + sanitized scene image prompt, plus “no text in image”
    - Truncates prompt to 4000 chars
    - Initializes `imageUrl: null`
  - Outputs a package object:
    - `story` (with prepared scenes)
    - `formData`, `storyId`, `totalScenes`, `createdAt`
- **Key variables/expressions:**
  - `$('Validate Input')` cross-node reference (requires that node executed in same run)
  - `response.choices[0].message.content`
- **Inputs/outputs:**
  - Input ← Generate Story (GPT-4)
  - Output → Generate Images (DALL-E 3)
- **Edge cases / failures:**
  - `JSON.parse` failure if OpenAI message content isn’t valid JSON.
  - If OpenAI response shape differs (no `choices[0]`), expressions fail.
  - Sanitization replaces quotes; could subtly alter intended prompt semantics.

---

### 2.5 Image Generation (DALL·E 3)
**Overview:** Iterates scenes, calls OpenAI Images API for each prompt, attaches `imageUrl` or a placeholder + error message.

**Nodes involved:**
- Sticky Note5
- **Generate Images (DALL-E 3)**

#### Generate Images (DALL-E 3)
- **Type / role:** `n8n-nodes-base.code` — custom HTTP calls with `fetch` to OpenAI Images.
- **Configuration choices:**
  - Uses hardcoded placeholder for API key:
    - `const OPENAI_API_KEY = 'YOUR_OPENAI_API_KEY';`
  - For each scene:
    - POST `https://api.openai.com/v1/images/generations`
    - Payload:
      - `model: 'dall-e-3'`
      - `prompt: scene.dallePrompt`
      - `n: 1`
      - `size: '1024x1024'`
      - `quality: 'standard'`
      - `response_format: 'url'`
    - If `result.error` exists, set:
      - `imageUrl` to `placehold.co` error image
      - `imageError` to message
    - Else set:
      - `imageUrl` to returned URL
      - `imageGeneratedAt`
  - Rate limiting: waits **1 second** between requests.
- **Inputs/outputs:**
  - Input ← Prepare Story Data
  - Output → Upload to Firebase Storage
- **Edge cases / failures:**
  - **Key issue:** this node does not use n8n OpenAI credentials; it requires manually setting `OPENAI_API_KEY`. If left unchanged, all image requests fail (401).
  - DALL·E 3 generation can be slow; sequential loop increases total runtime.
  - OpenAI can reject prompts (policy or safety) → result.error.
  - Node runtime limits on your n8n instance may be exceeded for 12 scenes.
- **Version notes:** Code node version `2`. Requires that `fetch` is available in the n8n runtime (it is in recent n8n builds).  

---

### 2.6 Firebase Storage Upload
**Overview:** Downloads each generated image and uploads it to Firebase Storage using the Storage REST endpoint, then rewrites `imageUrl` to the hosted Firebase URL.

**Nodes involved:**
- Sticky Note6
- **Upload to Firebase Storage**

#### Upload to Firebase Storage
- **Type / role:** `n8n-nodes-base.code` — custom download + upload via REST.
- **Configuration choices:**
  - Hardcoded bucket placeholder:
    - `const FIREBASE_BUCKET = 'your-project.appspot.com';`
  - For each scene:
    - Skip if `scene.imageError` or missing URL or placeholder URL
    - Download: `fetch(scene.imageUrl)` then `arrayBuffer()`
    - Upload:
      - POST `https://firebasestorage.googleapis.com/v0/b/{bucket}/o?name={encodedPath}`
      - `Content-Type: image/png`
      - Body: `Buffer.from(imageBuffer)`
    - Constructs Storage path:
      - `stories/{storyId}/scene_{i}.png`
  - Attempts to build `firebaseUrl` using `uploadResult.downloadTokens` (but see failure notes).
- **Inputs/outputs:**
  - Input ← Generate Images (DALL-E 3)
  - Output → Finalize Story
- **Edge cases / failures (important):**
  - **Authentication missing:** Upload call does not include Firebase auth (OAuth2 bearer token) or signed upload URL. In most Firebase projects, unauthenticated writes are denied → 401/403.
  - **Bug in code as provided:** the `firebaseUrl` assignment contains malformed template string / missing closing delimiter and a placeholder `YOUR_TOKEN_HERE`. As written, this code would throw a syntax error and fail node execution.
  - Even if upload succeeds, `downloadTokens` must be read properly and appended (often `?alt=media&token=...`).
  - Uploaded content type is always `image/png` but DALL·E URL may not be PNG; you may upload mismatched bytes.
  - No retry/backoff for transient network failures.
- **Version notes:** Code node version `2`.

---

### 2.7 Finalize Story Object
**Overview:** Removes internal helper properties and produces the final story object used for Firestore and the webhook response.

**Nodes involved:**
- Sticky Note7
- **Finalize Story**

#### Finalize Story
- **Type / role:** `n8n-nodes-base.code` — data cleanup and final shaping.
- **Configuration choices:**
  - Removes from each scene:
    - `sceneIndex`, `dallePrompt`, `imageError`
  - Constructs `finalStory`:
    - `id` = storyId
    - Story metadata (title, summaries, language, moral, etc.)
    - `imageStyle` from form
    - `scenes`, `characters`
    - `metadata` including createdAt/updatedAt, generator name, original prompt
  - Also computes a Firestore subcollection language key:
    - `language: data.formData.Language.toLowerCase().replace(' ', '_')`
- **Inputs/outputs:**
  - Input ← Upload to Firebase Storage
  - Output → Prepare Firestore Data
- **Edge cases / failures:**
  - If `data.formData.Language` contains multiple spaces, `.replace(' ', '_')` only replaces the first.
  - If earlier nodes failed and `data.story` is missing, throws.

---

### 2.8 Firestore Persistence
**Overview:** Converts the story object into Firestore REST “fields” format and writes it to a language-based collection path.

**Nodes involved:**
- Sticky Note8
- **Prepare Firestore Data**
- **Save to Firestore**

#### Prepare Firestore Data
- **Type / role:** `n8n-nodes-base.code` — converts JSON to Firestore REST document format.
- **Configuration choices:**
  - Hardcoded project ID placeholder:
    - `const FIREBASE_PROJECT_ID = 'your-project-id';`
  - `toFirestore(value)` converts:
    - boolean → `booleanValue`
    - integer → `integerValue` (string)
    - float → `doubleValue`
    - string → `stringValue`
    - array → `arrayValue.values`
    - object → `mapValue.fields`
    - null/undefined → `nullValue`
  - Builds `docFields` and sets:
    - `createdAt` as `timestampValue` (current time, not story metadata time)
  - Outputs:
    - `body: { fields: docFields }`
    - `collection: stories/{language}/items`
    - `docId: story.id`
    - `projectId`
    - `finalStory` (passed forward for response)
- **Inputs/outputs:**
  - Input ← Finalize Story
  - Output → Save to Firestore
- **Edge cases / failures:**
  - If `FIREBASE_PROJECT_ID` not set, Firestore request fails.
  - Firestore limits: document size (1 MiB) may be hit if scenes are long or include large arrays.
  - `createdAt` duplicated: story metadata also has createdAt; potential inconsistency.

#### Save to Firestore
- **Type / role:** `n8n-nodes-base.httpRequest` — Firestore REST API write.
- **Configuration choices:**
  - Method: `PATCH` (upsert-like behavior for a fixed doc ID)
  - URL expression:
    - `https://firestore.googleapis.com/v1/projects/{projectId}/databases/(default)/documents/{collection}/{docId}`
  - Authentication: **predefinedCredentialType** `googleApi`
  - JSON body: `={{ JSON.stringify($json.body) }}`
- **Inputs/outputs:**
  - Input ← Prepare Firestore Data
  - Output → Return Response
- **Edge cases / failures:**
  - Google credential missing scopes/permissions → 401/403.
  - Wrong collection path (nested collection under `stories/{language}/items`) must exist conceptually; Firestore will create documents but security rules may block.
  - Using `PATCH` without updateMask may overwrite fields depending on API behavior; typically it writes provided fields.

---

### 2.9 Webhook Response
**Overview:** Returns a JSON response containing the final story object.

**Nodes involved:**
- Sticky Note9
- **Return Response**

#### Return Response
- **Type / role:** `n8n-nodes-base.respondToWebhook` — sends HTTP response for the form/webhook execution.
- **Configuration choices:**
  - Response headers: `Content-Type: application/json`
  - Respond with: JSON
  - Response body includes:
    - `success: true`
    - message
    - `story`: pulled from `Prepare Firestore Data` node:
      - `{{ JSON.stringify($('Prepare Firestore Data').first().json.finalStory) }}`
- **Inputs/outputs:**
  - Input ← Save to Firestore
  - Output: ends workflow execution
- **Edge cases / failures:**
  - If Firestore save fails and workflow stops, no response may be sent unless n8n error handling is configured.
  - Double-encoding risk: `respondWith: json` but `story` is inserted via `JSON.stringify(...)`; this typically results in `story` being a **string** containing JSON, not an object. (Depends on how n8n evaluates the expression into the final JSON.)
  - Cross-node reference `$('Prepare Firestore Data')` requires that node executed successfully.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note | n8n-nodes-base.stickyNote | Documentation / canvas annotation | — | — | ## AI Illustrated Story Generator with DALL-E 3 & Firebase; Generate complete illustrated stories using AI... (setup steps, features) |
| Sticky Note1 | n8n-nodes-base.stickyNote | Documentation / canvas annotation | — | — | **Step 1: Form Input** Collects story topic, language, art style, number of scenes (1-12), audience, and mood. |
| Sticky Note2 | n8n-nodes-base.stickyNote | Documentation / canvas annotation | — | — | **Step 2: Validate** Ensures scene count is between 1-12. Generates unique story ID. |
| Sticky Note3 | n8n-nodes-base.stickyNote | Documentation / canvas annotation | — | — | **Step 3: Generate Story** GPT-4 creates complete narrative with scenes, characters, and image prompts. |
| Sticky Note4 | n8n-nodes-base.stickyNote | Documentation / canvas annotation | — | — | **Step 4: Prepare Data** Sanitizes text and optimizes prompts for DALL-E 3. |
| Sticky Note5 | n8n-nodes-base.stickyNote | Documentation / canvas annotation | — | — | **Step 5: DALL-E 3** Generates one image per scene. ~15 sec each. |
| Sticky Note6 | n8n-nodes-base.stickyNote | Documentation / canvas annotation | — | — | **Step 6: Upload** Downloads from DALL-E, uploads to Firebase Storage. |
| Sticky Note7 | n8n-nodes-base.stickyNote | Documentation / canvas annotation | — | — | **Step 7: Finalize** Cleans up data and prepares final story object. |
| Sticky Note8 | n8n-nodes-base.stickyNote | Documentation / canvas annotation | — | — | **Step 8: Firestore** Saves story to Firestore database. |
| Sticky Note9 | n8n-nodes-base.stickyNote | Documentation / canvas annotation | — | — | **Step 9: Response** Returns complete story JSON to the user. |
| Story Input Form | n8n-nodes-base.formTrigger | Collect user input; workflow entrypoint | — | Validate Input | **Step 1: Form Input** Collects story topic, language, art style, number of scenes (1-12), audience, and mood. |
| Validate Input | n8n-nodes-base.code | Clamp scenes; generate storyId | Story Input Form | Generate Story (GPT-4) | **Step 2: Validate** Ensures scene count is between 1-12. Generates unique story ID. |
| Generate Story (GPT-4) | n8n-nodes-base.httpRequest | OpenAI Chat Completions (GPT-4o) JSON story generation | Validate Input | Prepare Story Data | **Step 3: Generate Story** GPT-4 creates complete narrative with scenes, characters, and image prompts. |
| Prepare Story Data | n8n-nodes-base.code | Parse/validate GPT JSON; build DALL·E prompts | Generate Story (GPT-4) | Generate Images (DALL-E 3) | **Step 4: Prepare Data** Sanitizes text and optimizes prompts for DALL-E 3. |
| Generate Images (DALL-E 3) | n8n-nodes-base.code | Generate 1 image per scene via OpenAI Images | Prepare Story Data | Upload to Firebase Storage | **Step 5: DALL-E 3** Generates one image per scene. ~15 sec each. |
| Upload to Firebase Storage | n8n-nodes-base.code | Download DALL·E images; upload to Firebase Storage | Generate Images (DALL-E 3) | Finalize Story | **Step 6: Upload** Downloads from DALL-E, uploads to Firebase Storage. |
| Finalize Story | n8n-nodes-base.code | Remove internal fields; build final story object | Upload to Firebase Storage | Prepare Firestore Data | **Step 7: Finalize** Cleans up data and prepares final story object. |
| Prepare Firestore Data | n8n-nodes-base.code | Convert to Firestore REST fields; compute path | Finalize Story | Save to Firestore | **Step 8: Firestore** Saves story to Firestore database. |
| Save to Firestore | n8n-nodes-base.httpRequest | PATCH document to Firestore REST API | Prepare Firestore Data | Return Response | **Step 8: Firestore** Saves story to Firestore database. |
| Return Response | n8n-nodes-base.respondToWebhook | Return final JSON payload to user | Save to Firestore | — | **Step 9: Response** Returns complete story JSON to the user. |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow** in n8n.

2) **Add Form Trigger**
   - Node: **Form Trigger** (`Story Input Form`)
   - Set:
     - Form Title: `AI Story Generator`
     - Form Description: (as desired)
     - Add fields (required):
       - `Story Topic` (Textarea)
       - `Language` (Dropdown: English, Spanish, French, German, Portuguese, Italian, Japanese, Chinese, Korean, Dutch, Russian, Arabic)
       - `Image Style` (Dropdown: the 10 styles listed in the workflow)
       - `Number of Scenes` (Number)
       - `Target Audience` (Dropdown options listed)
       - `Story Mood` (Dropdown options listed)
   - Configure submit message: “Your story is being created! This takes 2-5 minutes. Please wait...”

3) **Add Code node: validation**
   - Node: **Code** (`Validate Input`)
   - Paste logic to:
     - Read form JSON
     - Parse and clamp `Number of Scenes` to 1..12 (default 4)
     - Generate `storyId`
     - Output merged object with `numScenes` and `storyId`

4) **Connect** `Story Input Form` → `Validate Input`.

5) **Add HTTP Request node: OpenAI Chat Completions**
   - Node: **HTTP Request** (`Generate Story (GPT-4)`)
   - Method: `POST`
   - URL: `https://api.openai.com/v1/chat/completions`
   - Authentication: **OpenAI API credential** (n8n credential type: `openAiApi`)
   - Body type: JSON
   - Set body to include:
     - `model: gpt-4o`
     - messages (system + user), where user message interpolates:
       - topic, language, numScenes, audience, mood, style
     - `response_format` set to `json_schema` with strict schema for story/scenes/characters (matching the workflow’s required fields)

6) **Connect** `Validate Input` → `Generate Story (GPT-4)`.

7) **Add Code node: prompt preparation**
   - Node: **Code** (`Prepare Story Data`)
   - Implement:
     - Parse `response.choices[0].message.content` as JSON
     - Sanitize text
     - For each scene, generate `dallePrompt` (prepend chosen style + “children’s book illustration”, append “no text”)
     - Add helper fields: `sceneIndex`, `imageUrl: null`
     - Output: `{ story, formData, storyId, totalScenes, createdAt }`
   - Ensure it references prior node data:
     - `$('Validate Input').first().json`

8) **Connect** `Generate Story (GPT-4)` → `Prepare Story Data`.

9) **Add Code node: DALL·E generation**
   - Node: **Code** (`Generate Images (DALL-E 3)`)
   - Use `fetch` to call:
     - `POST https://api.openai.com/v1/images/generations`
   - Provide:
     - `model: dall-e-3`, `prompt`, `n:1`, `size:1024x1024`, `quality:standard`, `response_format:url`
   - Add basic error handling + placeholder image URL
   - Add a 1s delay between calls
   - **Credential setup:** Replace `OPENAI_API_KEY` with your key **or** refactor the node to use an n8n credential/HTTP Request node (recommended) so secrets aren’t hardcoded.

10) **Connect** `Prepare Story Data` → `Generate Images (DALL-E 3)`.

11) **Add Code node: Firebase Storage upload**
   - Node: **Code** (`Upload to Firebase Storage`)
   - Set `FIREBASE_BUCKET` to your bucket (e.g. `my-project.appspot.com`)
   - For each scene:
     - Download the DALL·E `imageUrl`
     - Upload bytes to Firebase Storage path `stories/{storyId}/scene_{i}.png`
     - Replace scene `imageUrl` with Firebase-hosted URL
   - **Credential setup:** implement authenticated upload (OAuth2 bearer token or signed URL) depending on your Firebase security rules.
   - **Important:** Fix the malformed `firebaseUrl` string construction and handle `downloadTokens` correctly if you rely on token-based access.

12) **Connect** `Generate Images (DALL-E 3)` → `Upload to Firebase Storage`.

13) **Add Code node: Finalize**
   - Node: **Code** (`Finalize Story`)
   - Remove internal fields (`sceneIndex`, `dallePrompt`, `imageError`)
   - Construct `finalStory` with `metadata`
   - Output `{ finalStory, language }` where `language` is a normalized key

14) **Connect** `Upload to Firebase Storage` → `Finalize Story`.

15) **Add Code node: Prepare Firestore REST document**
   - Node: **Code** (`Prepare Firestore Data`)
   - Set `FIREBASE_PROJECT_ID`
   - Convert `finalStory` to Firestore REST `fields` format via `toFirestore()`
   - Output:
     - `projectId`, `collection` (e.g. `stories/{language}/items`), `docId`, `body`, and `finalStory`

16) **Connect** `Finalize Story` → `Prepare Firestore Data`.

17) **Add HTTP Request node: Save to Firestore**
   - Node: **HTTP Request** (`Save to Firestore`)
   - Method: `PATCH`
   - URL:
     - `https://firestore.googleapis.com/v1/projects/{{ $json.projectId }}/databases/(default)/documents/{{ $json.collection }}/{{ $json.docId }}`
   - Auth: **Google API credential** (`googleApi`) with access to Firestore
   - Body: JSON = `$json.body`

18) **Connect** `Prepare Firestore Data` → `Save to Firestore`.

19) **Add Respond to Webhook node**
   - Node: **Respond to Webhook** (`Return Response`)
   - Respond with: JSON
   - Header: `Content-Type: application/json`
   - Body should include the `finalStory`
   - Recommendation: return the object directly (avoid `JSON.stringify` inside a JSON response) to prevent double-encoding.

20) **Connect** `Save to Firestore` → `Return Response`.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “AI Illustrated Story Generator with DALL-E 3 & Firebase” description, setup steps, supported languages/styles/scenes, and Firebase + Firestore storage design | From the main sticky note (workflow canvas annotation) |
| Setup mentions updating variables in Code nodes: `OPENAI_API_KEY`, `FIREBASE_BUCKET`, `FIREBASE_PROJECT_ID` | Ensure these are not left as placeholders; prefer n8n credentials over hardcoding secrets |

