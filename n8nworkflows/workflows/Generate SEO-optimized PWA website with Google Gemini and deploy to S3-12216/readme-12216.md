Generate SEO-optimized PWA website with Google Gemini and deploy to S3

https://n8nworkflows.xyz/workflows/generate-seo-optimized-pwa-website-with-google-gemini-and-deploy-to-s3-12216


# Generate SEO-optimized PWA website with Google Gemini and deploy to S3

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:** Generate a complete **SEO-optimized, multi-language, PWA-ready single-page website** using **Google Gemini (via LangChain nodes)**, then **deploy all files to AWS S3** and **email the user** the live URL plus PWA install instructions.

**Target use cases:**
- Rapid generation of marketing/landing/corporate websites with multi-language support
- Basic PWA enablement (manifest + service worker)
- Basic SEO package generation (meta tags, Open Graph, Schema.org JSON-LD, sitemap, robots)

**Logical blocks:**
1.1 **Input Reception (Form Trigger)**  
1.2 **Normalization & Configuration (Form → structured JSON)**  
1.3 **AI SEO Metadata Generation (Gemini)**  
1.4 **AI Website HTML Generation (Gemini)**  
1.5 **PWA + SEO Files Generation (manifest/sw/sitemap/robots)**  
1.6 **File Assembly & Packaging (inject links, generate bundle)**  
1.7 **Deployment to S3 (binary conversion + upload)**  
1.8 **Success Response & Email Notification (URL build + Gmail)**

---

## 2. Block-by-Block Analysis

### 2.1 Input Reception (Form Trigger)

**Overview:** Presents a public n8n form for users to enter website requirements. Submits data into the workflow and immediately responds with a “processing” message.

**Nodes involved:**
- **Website Request Form**

**Node details**
- **Website Request Form**
  - **Type / role:** `n8n-nodes-base.formTrigger` — entry point collecting structured user input.
  - **Configuration choices:**
    - Form title: “AI Multi-Language PWA Website Generator”
    - Respond message: “Your multi-language PWA website is being generated with AI...”
    - Fields include: Website Name, Primary Purpose (dropdown), Target Languages (comma-separated), Design Style (dropdown), SEO Keywords, Main Heading, Business Description, Contact Email, optional company/phone/social links, GA ID, FB Pixel ID, special features.
  - **Inputs/outputs:** No inputs (trigger). Output is one JSON item containing form field labels as keys.
  - **Edge cases / failures:**
    - User enters invalid language codes or extra spaces (handled later via trimming; not validated against ISO lists).
    - Missing required fields is prevented by form configuration, but malformed email still possible depending on n8n validation.
  - **Version notes:** Node is **typeVersion 2.2**; behavior/options may vary with older n8n.

---

### 2.2 Normalization & Configuration (Form → structured JSON)

**Overview:** Converts form submission keys into a consistent internal data model, extracts language list, parses social links, and generates a URL-safe slug and timestamp.

**Nodes involved:**
- **Process Form Data**

**Node details**
- **Process Form Data**
  - **Type / role:** `n8n-nodes-base.code` — transforms input JSON to a normalized schema.
  - **Key configuration/logic:**
    - `langs`: splits `"Target Languages"` by comma, trims, defaults to `"en"`.
    - `socialLinks`: parses `"Social Media Links"` by comma; detects platform via substring matching (`facebook`, `twitter`/`x.com`, `linkedin`, `instagram`).
    - `slug`: lowercases Website Name and replaces non-alphanumerics with `-`, collapses repeats, trims edges.
    - Creates a consistent output object:
      - `websiteName`, `purpose`, `languages`, `primaryLang` (= first language), `designStyle`, `seoKeywords`, `mainHeading`, `description`,
      - `contactEmail`, `companyName` (fallback to websiteName), `phone`, `socialLinks`,
      - `gaId`, `fbPixelId`, `specialFeatures`, `slug`, `timestamp`.
  - **Inputs/outputs:**
    - Input: raw form JSON from **Website Request Form**
    - Output: single normalized JSON item
    - Downstream: feeds **Generate SEO Metadata**
  - **Edge cases / failures:**
    - If Website Name is empty (should be required), slug becomes empty → downstream cache name and filenames can break.
    - `companyName.substring(0,12)` later (PWA manifest) assumes `companyName` is a string (it is, by fallback).
    - Social parsing is simplistic; non-standard URLs won’t be classified.

---

### 2.3 AI SEO Metadata Generation (Gemini)

**Overview:** Uses Gemini to generate structured SEO metadata as **strict JSON** (meta title/description, Open Graph fields, Schema.org JSON-LD, keyword list, heading suggestions).

**Nodes involved:**
- **Generate SEO Metadata**
- **Google Gemini Chat Model**

**Node details**
- **Google Gemini Chat Model**
  - **Type / role:** `@n8n/n8n-nodes-langchain.lmChatGoogleGemini` — provides the LLM backend for LangChain chain nodes.
  - **Configuration choices:** Default options (no explicit model/temperature shown).
  - **Connections:** Connected to **Generate SEO Metadata** via the `ai_languageModel` port.
  - **Edge cases / failures:** Credential/auth errors, quota limits, model refusal, latency/timeouts.

- **Generate SEO Metadata**
  - **Type / role:** `@n8n/n8n-nodes-langchain.chainLlm` — runs a prompt against the connected Gemini model.
  - **Configuration choices:**
    - Prompt instructs: “Respond ONLY in valid JSON with these keys: metaTitle, metaDescription, ogTitle, ogDescription, schemaType, schemaJson (string), keywords (array), h1Suggestion, h2Suggestions (array).”
    - Interpolates:
      - `{{ $json.companyName }}`, `{{ $json.purpose }}`, `{{ $json.seoKeywords }}`, `{{ $json.description }}`, `{{ $json.languages.join(', ') }}`
  - **Inputs/outputs:**
    - Input: normalized JSON from **Process Form Data**
    - Output: LLM response in `item.json.text` (typical for this node), passed to **Generate Website HTML**
  - **Edge cases / failures:**
    - Model may output non-JSON or JSON wrapped in markdown → downstream HTML prompt still includes it, but you lose structured parsing. (This workflow does **not** validate or parse the SEO JSON; it passes it through as text.)
    - If `schemaJson` is requested “as a string”, Gemini may return an object instead; again not validated here.

---

### 2.4 AI Website HTML Generation (Gemini)

**Overview:** Generates a complete single HTML document with embedded CSS/JS, SEO tags, structured data, multi-language UI elements, hreflang tags, and PWA hooks.

**Nodes involved:**
- **Generate Website HTML**
- **Google Gemini Chat Model1**

**Node details**
- **Google Gemini Chat Model1**
  - **Type / role:** `@n8n/n8n-nodes-langchain.lmChatGoogleGemini` — second Gemini model node used for the HTML generation chain.
  - **Configuration choices:** Default options.
  - **Connections:** Connected to **Generate Website HTML** via `ai_languageModel`.
  - **Edge cases / failures:** Same as prior model node (auth/quota/refusal/timeout).

- **Generate Website HTML**
  - **Type / role:** `@n8n/n8n-nodes-langchain.chainLlm` — prompts Gemini to generate full production-ready HTML.
  - **Configuration choices (interpreted):**
    - Pulls website requirements from **Process Form Data** explicitly using:
      - `{{ $('Process Form Data').item.json.websiteName }}`, etc.
    - Injects SEO block from upstream node using:
      - `{{ $json.text }}` (this refers to the output of **Generate SEO Metadata**)
    - Enforces constraints:
      - Return only HTML from `<!DOCTYPE html>` to `</html>`
      - No external CDN links
      - Include `<link rel="manifest" href="manifest.json">`
      - Register service worker before `</body>`
      - Include GA/Pixel if IDs are provided
      - Add language selector and hreflang tags
  - **Inputs/outputs:**
    - Input: output of **Generate SEO Metadata** (plus it reaches back to Process Form Data via expressions)
    - Output: HTML in `item.json.text`
    - Downstream: **Generate PWA Components**
  - **Edge cases / failures:**
    - Model may still wrap HTML in triple backticks; handled later by “Assemble All Files”.
    - HTML may omit requested parts (manifest link, SW registration) — later code injects minimal hooks if missing.
    - Multi-language requirement: model might not generate separate pages per language; this workflow ultimately deploys a single HTML file plus shared PWA assets.

---

### 2.5 PWA + SEO Files Generation (manifest/sw/sitemap/robots)

**Overview:** Creates `manifest.json`, `sw.js`, `sitemap.xml`, and `robots.txt` based on processed form inputs and design style, with basic caching and SEO discovery support.

**Nodes involved:**
- **Generate PWA Components**

**Node details**
- **Generate PWA Components**
  - **Type / role:** `n8n-nodes-base.code` — deterministic generation of PWA and SEO support files.
  - **Key configuration/logic:**
    - Theme/background colors derived from `designStyle` heuristics:
      - Dark → `#111827` / bg `#000000`
      - Tech → `#4F46E5`
      - Creative → `#7C3AED`
      - Default → `#3B82F6`
    - `manifest.json` includes name, short_name, description, start_url `/`, standalone display, icons `icon-192.png` and `icon-512.png` (note: these icon files are **not generated or uploaded** in this workflow).
    - `sw.js`: basic cache-first for `/` and `manifest.json`, with cache version `${slug}-v1`.
    - `sitemap.xml` and `robots.txt` contain placeholder domain `https://YOUR_DOMAIN/...`
  - **Inputs/outputs:**
    - Inputs: HTML generation triggers this node; it reads **Process Form Data** via expression.
    - Output: JSON with keys `{ manifest, sw, sitemap, robots }` to **Assemble All Files**.
  - **Edge cases / failures:**
    - `YOUR_DOMAIN` placeholders make sitemap/robots inaccurate unless replaced (this workflow does not replace them).
    - Missing icon assets referenced by manifest may cause PWA install warnings.
    - Service worker caches only `/` and `manifest.json`—the HTML file is not necessarily `/` unless the S3 site is configured accordingly.

---

### 2.6 File Assembly & Packaging

**Overview:** Cleans the LLM HTML output, injects missing manifest/service worker registration if needed, and emits a list of deployable files (HTML + PWA + SEO files) as individual items.

**Nodes involved:**
- **Assemble All Files**

**Node details**
- **Assemble All Files**
  - **Type / role:** `n8n-nodes-base.code` — converts generated content into a deployable file bundle.
  - **Key configuration/logic:**
    - Retrieves:
      - `htmlRaw` from **Generate Website HTML**: `$('Generate Website HTML').item.json.text`
      - `pwa` from current input `$json` (output of Generate PWA Components)
      - `pd` from **Process Form Data**
    - Strips markdown fences (```html / ```) and trims.
    - Attempts to cut leading text before `<!doctype html>` if present.
    - Ensures `<link rel="manifest"...>` exists; injects after `<head>` if missing.
    - Ensures service worker registration script exists; injects before `</body>` if missing.
    - Constructs main HTML filename: `slug + '-' + timestamp + '.html'`
    - Returns **multiple output items** (one per file):
      - HTML file, `manifest.json`, `sw.js`, `sitemap.xml`, `robots.txt`
  - **Inputs/outputs:**
    - Input: PWA components JSON (and accesses HTML + process data through expressions)
    - Output: array of items `{fileName, content, contentType}` to **Convert to Binary**
  - **Edge cases / failures:**
    - Injection relies on exact `<head>` and `</body>` substrings; if model produces uppercase tags or atypical formatting, replace may fail silently.
    - Filename collisions are unlikely due to timestamp, but possible under high concurrency with identical timestamps (rare).
    - The main HTML file is not named `index.html`, which affects static hosting default behavior unless you configure S3 routing or reference the exact file.

---

### 2.7 Deployment to S3 (binary conversion + upload)

**Overview:** Converts each file item to binary and uploads them to a specified S3 bucket.

**Nodes involved:**
- **Convert to Binary**
- **Upload to S3**

**Node details**
- **Convert to Binary**
  - **Type / role:** `n8n-nodes-base.moveBinaryData` — creates a binary field for S3 upload.
  - **Configuration choices:**
    - Mode: `jsonToBinary`
    - Destination binary key: `file`
    - Encoding: `utf8`
    - File name: `={{ $json.fileName }}`
    - MIME type: `={{ $json.contentType }}`
  - **Inputs/outputs:**
    - Input: each file item from Assemble All Files
    - Output: same item with `binary.file` attached
  - **Edge cases / failures:**
    - If `content` is undefined/empty, binary may be empty but still uploads.
    - Very large HTML could hit n8n memory limits (rare for single-page output but possible).

- **Upload to S3**
  - **Type / role:** `n8n-nodes-base.awsS3` — uploads each binary file as an object.
  - **Configuration choices:**
    - Operation: Upload
    - Bucket: `YOUR_S3_BUCKET_NAME` (placeholder; must be replaced)
    - Binary Property Name: `file`
    - File Name (object key): `={{ $json.fileName }}`
  - **Inputs/outputs:**
    - Input: binary file item
    - Output: S3 upload result item (used by Build Response)
  - **Edge cases / failures:**
    - Credentials/permissions: requires `s3:PutObject` and likely `s3:PutObjectAcl` depending on bucket policy.
    - Bucket region mismatch or missing bucket.
    - Static website hosting requires public access configuration or CloudFront; otherwise URL may not be accessible publicly.

---

### 2.8 Success Response & Email Notification

**Overview:** Builds a final “success” object with a computed website URL and sends an HTML email to the requester.

**Nodes involved:**
- **Build Response**
- **Send Confirmation Email**

**Node details**
- **Build Response**
  - **Type / role:** `n8n-nodes-base.code` — aggregates uploaded items and constructs a final response JSON.
  - **Key configuration/logic:**
    - `items = $input.all()` to collect all S3 upload results
    - `baseUrl = 'https://YOUR_S3_BUCKET_NAME.s3.YOUR_REGION.amazonaws.com'` (placeholder)
    - Finds main HTML file via `endsWith('.html')`
    - Returns:
      - `status: 'success'`
      - `websiteUrl`: baseUrl + '/' + html filename (or baseUrl)
      - `filesUploaded`: count
      - `timestamp`
  - **Inputs/outputs:**
    - Input: results from **Upload to S3**
    - Output: single JSON object to **Send Confirmation Email**
  - **Edge cases / failures:**
    - URL format may be wrong for your bucket/region, or if you use S3 static website endpoint vs REST endpoint.
    - If no `.html` file is found (unexpected), websiteUrl becomes baseUrl only.

- **Send Confirmation Email**
  - **Type / role:** `n8n-nodes-base.gmail` — sends confirmation email with the generated site link.
  - **Configuration choices:**
    - To: `={{ $('Process Form Data').item.json.contactEmail }}`
    - Subject: “Your Multi-Language PWA Website is Live!”
    - HTML message includes:
      - Website name and language list
      - Mentions PWA install
      - Conditional bullet for GA tracking: `{{ ...gaId ? '<li>Google Analytics tracking</li>' : '' }}`
      - Button linking to `{{ $json.websiteUrl }}` (from Build Response)
  - **Inputs/outputs:**
    - Input: Build Response output (for `websiteUrl`)
    - Also references Process Form Data for personalization
  - **Edge cases / failures:**
    - Gmail OAuth credentials required; may fail due to missing scopes or Google security restrictions.
    - If the workflow runs long, Gmail send could still succeed but user may click link before objects are publicly readable.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Workflow Overview | n8n-nodes-base.stickyNote | Documentation / canvas annotation |  |  | ## AI Multi-Language PWA Website Generator with SEO Optimization … ⚠️ Note: Requires OpenAI or Google Gemini API key, AWS S3 bucket, and Gmail credentials. |
| Step 1 - Trigger & Config | n8n-nodes-base.stickyNote | Documentation / block label |  |  | ## Trigger & Configuration Collects detailed website requirements… |
| Step 2 - AI Generation | n8n-nodes-base.stickyNote | Documentation / block label |  |  | ## AI SEO & Content Generation Two-stage AI pipeline… |
| Step 3 - PWA & Files | n8n-nodes-base.stickyNote | Documentation / block label |  |  | ## PWA Processing & File Assembly Generates PWA components… |
| Step 4 - Deploy & Notify | n8n-nodes-base.stickyNote | Documentation / block label |  |  | ## Deploy & Notify Splits the file bundle… uploads all files to S3… sends … email… |
| Website Request Form | n8n-nodes-base.formTrigger | Entry point: collect website requirements | — | Process Form Data | ## Trigger & Configuration Collects detailed website requirements… |
| Process Form Data | n8n-nodes-base.code | Normalize/structure form data (langs, slug, IDs) | Website Request Form | Generate SEO Metadata | ## Trigger & Configuration Collects detailed website requirements… |
| Google Gemini Chat Model | @n8n/n8n-nodes-langchain.lmChatGoogleGemini | LLM backend for SEO metadata generation | — | Generate SEO Metadata (ai_languageModel) | ## AI SEO & Content Generation Two-stage AI pipeline… |
| Generate SEO Metadata | @n8n/n8n-nodes-langchain.chainLlm | Generate SEO JSON metadata (prompted) | Process Form Data (+ model) | Generate Website HTML | ## AI SEO & Content Generation Two-stage AI pipeline… |
| Google Gemini Chat Model1 | @n8n/n8n-nodes-langchain.lmChatGoogleGemini | LLM backend for HTML generation | — | Generate Website HTML (ai_languageModel) | ## AI SEO & Content Generation Two-stage AI pipeline… |
| Generate Website HTML | @n8n/n8n-nodes-langchain.chainLlm | Generate complete HTML with SEO+PWA hooks | Generate SEO Metadata (+ model) | Generate PWA Components | ## AI SEO & Content Generation Two-stage AI pipeline… |
| Generate PWA Components | n8n-nodes-base.code | Create manifest, service worker, sitemap, robots | Generate Website HTML | Assemble All Files | ## PWA Processing & File Assembly Generates PWA components… |
| Assemble All Files | n8n-nodes-base.code | Clean HTML, inject missing hooks, output file items | Generate PWA Components | Convert to Binary | ## PWA Processing & File Assembly Generates PWA components… |
| Convert to Binary | n8n-nodes-base.moveBinaryData | Convert file JSON to binary for upload | Assemble All Files | Upload to S3 | ## Deploy & Notify Splits the file bundle… uploads all files to S3… |
| Upload to S3 | n8n-nodes-base.awsS3 | Upload each file to S3 bucket | Convert to Binary | Build Response | ## Deploy & Notify Splits the file bundle… uploads all files to S3… |
| Build Response | n8n-nodes-base.code | Compute final URL + status after uploads | Upload to S3 | Send Confirmation Email | ## Deploy & Notify Splits the file bundle… sends … email… |
| Send Confirmation Email | n8n-nodes-base.gmail | Email user with link + PWA install info | Build Response | — | ## Deploy & Notify Splits the file bundle… sends … email… |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow** in n8n.

2) **Add trigger: “Form Trigger” node**
   - Name: **Website Request Form**
   - Form title: *AI Multi-Language PWA Website Generator*
   - Description: *Create a professional, SEO-optimized, multi-language Progressive Web App with AI.*
   - Add fields (mark required where indicated):
     - Website Name (required)
     - Primary Purpose (dropdown; required; options: Corporate Website, Portfolio, Product Landing Page, SaaS Landing, Blog, Event Page, Restaurant or Local Business, Non-profit)
     - Target Languages (required; placeholder like `en, ja, es, fr`)
     - Design Style (dropdown; required; options: Modern Minimalist, Corporate Professional, Creative Bold, Tech SaaS, Dark Mode, Colorful Playful)
     - SEO Keywords (required)
     - Main Heading (required)
     - Business Description (textarea; required)
     - Contact Email (email; required)
     - Company Name (optional)
     - Phone Number (optional)
     - Social Media Links (optional, comma-separated URLs)
     - Google Analytics ID (optional)
     - Facebook Pixel ID (optional)
     - Special Features (textarea; optional)
   - Response text on submit: *Your multi-language PWA website is being generated with AI...*

3) **Add “Code” node** after the form
   - Name: **Process Form Data**
   - Paste logic to:
     - Split Target Languages by commas, trim
     - Parse Social Media Links into `socialLinks` object
     - Create `slug` from Website Name
     - Output normalized keys listed in section 2.2

4) **Add Gemini model node (LangChain)**
   - Name: **Google Gemini Chat Model**
   - Configure credentials: **Google Gemini / Google AI Studio API key** (as required by your n8n LangChain Gemini node)
   - Keep default options unless you want to set temperature/max tokens.

5) **Add “Chain LLM” node for SEO**
   - Name: **Generate SEO Metadata**
   - Connect:
     - Main input from **Process Form Data**
     - `ai_languageModel` input from **Google Gemini Chat Model**
   - Prompt: instruct the model to output **ONLY valid JSON** with keys: metaTitle, metaDescription, ogTitle, ogDescription, schemaType, schemaJson (string), keywords (array), h1Suggestion, h2Suggestions (array). Use expressions for business/purpose/keywords/description/languages.

6) **Add second Gemini model node**
   - Name: **Google Gemini Chat Model1**
   - Use same (or different) Gemini credentials and options.

7) **Add “Chain LLM” node for HTML**
   - Name: **Generate Website HTML**
   - Connect:
     - Main input from **Generate SEO Metadata**
     - `ai_languageModel` input from **Google Gemini Chat Model1**
   - Prompt: request a complete HTML document only, and include:
     - SEO tags + Schema.org JSON-LD in head
     - hreflang links
     - language selector
     - `<link rel="manifest" href="manifest.json">`
     - service worker registration JS
     - optional GA and FB Pixel if IDs exist
     - no external CDN links

8) **Add “Code” node for PWA/SEO files**
   - Name: **Generate PWA Components**
   - Generate:
     - `manifest.json` (as stringified JSON)
     - `sw.js`
     - `sitemap.xml` (note: decide your real domain instead of `YOUR_DOMAIN`)
     - `robots.txt`

9) **Add “Code” node to assemble files**
   - Name: **Assemble All Files**
   - Read:
     - HTML from **Generate Website HTML** (`item.json.text`)
     - PWA content from current input
     - Processed data from **Process Form Data**
   - Clean markdown fences, ensure manifest link + SW registration, emit **multiple items** each with:
     - `fileName`, `content`, `contentType`
   - Use timestamped HTML filename (or change to `index.html` if desired for S3 hosting).

10) **Add “Move Binary Data” node**
   - Name: **Convert to Binary**
   - Mode: **JSON to Binary**
   - Destination key: `file`
   - Encoding: `utf8`
   - File name expression: `{{ $json.fileName }}`
   - MIME type expression: `{{ $json.contentType }}`

11) **Add “AWS S3” node**
   - Name: **Upload to S3**
   - Credentials: configure AWS access key/secret (or IAM role) with permission to upload objects.
   - Operation: **Upload**
   - Bucket Name: set your real bucket (replace `YOUR_S3_BUCKET_NAME`)
   - Binary Property: `file`
   - File Name: `{{ $json.fileName }}`

12) **Add “Code” node to build final response**
   - Name: **Build Response**
   - Aggregate all inputs (`$input.all()`), compute:
     - `baseUrl` (choose correct S3 URL format for your setup)
     - `websiteUrl` pointing to the uploaded `.html`
     - `filesUploaded`, `timestamp`

13) **Add “Gmail” node**
   - Name: **Send Confirmation Email**
   - Credentials: Gmail OAuth2 in n8n (ensure appropriate scopes for sending mail).
   - To: `{{ $('Process Form Data').item.json.contactEmail }}`
   - Subject: “Your Multi-Language PWA Website is Live!”
   - Message: HTML body that links to `{{ $json.websiteUrl }}` and includes PWA install instructions.

14) **Connect nodes in order (main path):**  
   Website Request Form → Process Form Data → Generate SEO Metadata → Generate Website HTML → Generate PWA Components → Assemble All Files → Convert to Binary → Upload to S3 → Build Response → Send Confirmation Email

15) **Connect model ports:**  
   - Google Gemini Chat Model → Generate SEO Metadata (`ai_languageModel`)  
   - Google Gemini Chat Model1 → Generate Website HTML (`ai_languageModel`)

16) **Finalize placeholders**
   - Replace:
     - `YOUR_S3_BUCKET_NAME` in **Upload to S3**
     - `YOUR_S3_BUCKET_NAME` and `YOUR_REGION` in **Build Response**
     - `YOUR_DOMAIN` inside sitemap/robots generation (or compute from bucket/CloudFront domain)

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “Requires OpenAI or Google Gemini API key, AWS S3 bucket, and Gmail credentials.” | From the workflow overview sticky note |
| S3 static hosting must be enabled and objects must be publicly readable (or served via CloudFront) for the emailed URL to work. | Deployment / access consideration |
| `sitemap.xml` and `robots.txt` contain `YOUR_DOMAIN` placeholders and should be replaced with the actual public domain. | SEO correctness |
| `manifest.json` references `icon-192.png` and `icon-512.png` but the workflow does not generate/upload these assets. | PWA completeness consideration |
| Main HTML is uploaded as `slug-timestamp.html` (not `index.html`), so your hosting entry URL must point to that file (or change naming). | Hosting behavior consideration |