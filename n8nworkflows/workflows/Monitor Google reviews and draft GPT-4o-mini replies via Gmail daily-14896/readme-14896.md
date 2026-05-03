Monitor Google reviews and draft GPT-4o-mini replies via Gmail daily

https://n8nworkflows.xyz/workflows/monitor-google-reviews-and-draft-gpt-4o-mini-replies-via-gmail-daily-14896


# Monitor Google reviews and draft GPT-4o-mini replies via Gmail daily

Now I have a thorough understanding of the workflow. Let me write the complete reference document. Workflow Reference: Monitor Google Reviews and Draft GPT-4o-mini Replies via Gmail Daily

---

## 1. Workflow Overview

This workflow automates daily monitoring of Google reviews for a local business. Each morning at 9:00 AM it fetches the latest reviews from the Google Places API, classifies them as positive or negative, uses OpenAI GPT-4o-mini to generate personalised reply drafts for every review, and delivers a formatted digest email via Gmail to the business owner. The owner can then copy-paste the drafts directly into Google Business Profile.

**Target Use Cases:** Local business owners, agencies, and reputation managers who need a hands-free daily review digest with AI-crafted response drafts ready to publish.

**Logical Blocks:**

| Block | Purpose |
|---|---|
| 1.1 Schedule and Config | Daily cron trigger at 9 AM; stores all configurable values (Place ID, API key, business name, recipient email, owner name) in one Set node used throughout |
| 1.2 Review Fetch and Classification | HTTP call to Google Places Details API; Code node parses the response, splits reviews into positive (4–5 stars) and negative (1–3 stars), builds a text summary; IF node guards against empty review sets |
| 1.3 AI Reply Generation | LangChain Agent with GPT-4o-mini receives the full review summary and writes a personalised reply draft per review, following distinct guidelines for positive and negative sentiment |
| 1.4 Email Delivery | Set node assembles email fields (recipient, subject, body); Gmail node sends a daily digest with all drafts and posting instructions |

**Single entry point:** Schedule trigger.  
**Single exit point:** Gmail send (or workflow stops at the IF node if no reviews exist).

---

## 2. Block-by-Block Analysis

### Block 1.1 — Schedule and Config

**Overview:** A cron-based schedule trigger fires the workflow once daily. A Set node immediately stores all runtime configuration—Google Place ID, API key, business name, owner email, and owner name—so downstream nodes reference a single source of truth.

**Nodes Involved:**
- 1. Schedule — Every Day 9AM
- 2. Set — Config Values

**Node Details:**

#### 1. Schedule — Every Day 9AM
- **Type:** `n8n-nodes-base.scheduleTrigger` (v1.2)
- **Technical Role:** Workflow entry point; emits one empty item per execution.
- **Configuration:** Cron expression `0 9 * * *` (every day at 09:00 server time).
- **Key expressions:** None.
- **Input:** None (trigger node).
- **Output:** Connects to "2. Set — Config Values".
- **Edge cases:** Server timezone differences may shift the effective local time; workflow must be **activated** (not just saved) for the cron to run.

#### 2. Set — Config Values
- **Type:** `n8n-nodes-base.set` (v3.4)
- **Technical Role:** Central configuration store; all downstream nodes reference these values.
- **Configuration — Assignments:**

| Name | Type | Default Value | Purpose |
|---|---|---|---|
| `placeId` | string | `YOUR_GOOGLE_PLACE_ID` | Google Place ID for the business |
| `apiKey` | string | `YOUR_GOOGLE_PLACES_API_KEY` | Google Places API key |
| `businessName` | string | `YOUR BUSINESS NAME` | Display name used in prompts and email |
| `ownerEmail` | string | `user@example.com` | Recipient email for the daily digest |
| `ownerName` | string | `YOUR NAME` | Sign-off name in AI-generated replies |

- **Key expressions:** None at this node; values are static placeholders the user must replace.
- **Input:** From "1. Schedule — Every Day 9AM".
- **Output:** Connects to "3. HTTP — Fetch Google Places Reviews".
- **Edge cases:** Leaving any placeholder value unchanged will cause downstream failures (API 401/403, misaddressed email, generic AI replies). The Place ID must be for a location that has the Places API enabled in Google Cloud Console.

---

### Block 1.2 — Review Fetch and Classification

**Overview:** An HTTP Request node calls the Google Places Details API to retrieve reviews. A Code node parses the JSON, separates positive (4–5 stars) from negative (1–3 stars), and builds a plain-text review summary. An IF node checks whether any reviews exist; the workflow stops if none are found.

**Nodes Involved:**
- 3. HTTP — Fetch Google Places Reviews
- 4. Code — Parse and Classify Reviews
- 5. IF — Are There Reviews?

**Node Details:**

#### 3. HTTP — Fetch Google Places Reviews
- **Type:** `n8n-nodes-base.httpRequest` (v4.2)
- **Technical Role:** External API call to Google Places Details endpoint.
- **Configuration:**
  - **Method:** GET (default)
  - **URL (expression):** `https://maps.googleapis.com/maps/api/place/details/json?place_id={{ $json.placeId }}&fields=name,rating,reviews&key={{ $json.apiKey }}`
  - **Response format:** JSON (default)
  - **Authentication:** Via API key in query string (no OAuth).
- **Key expressions:** `$json.placeId` and `$json.apiKey` from the upstream Set node.
- **Input:** From "2. Set — Config Values".
- **Output:** Connects to "4. Code — Parse and Classify Reviews".
- **Edge cases:**
  - Invalid or revoked API key → HTTP 403 or `REQUEST_DENIED` status in response body.
  - Wrong Place ID → `NOT_FOUND` status.
  - Google API quota exhausted → HTTP 429.
  - Network timeout on Google's side (default n8n timeout applies).
  - The API returns up to 5 reviews by default; for businesses with many reviews, only the most recent subset is returned.

#### 4. Code — Parse and Classify Reviews
- **Type:** `n8n-nodes-base.code` (v2)
- **Technical Role:** Transforms raw Google Places JSON into structured output with sentiment classification and a formatted text summary.
- **Configuration:** Single JavaScript code block (no mode toggle needed).
- **Key logic:**
  1. Reads `$input.first().json` for the API response and references `$('2. Set — Config Values').item.json` for config values.
  2. Extracts `result.reviews` array; if empty, returns `{ hasReviews: false, message: 'No reviews found…', reviews: [] }`.
  3. Filters into `positiveReviews` (rating ≥ 4) and `negativeReviews` (rating ≤ 3).
  4. Builds `reviewSummary` string with each review numbered, tagged as POSITIVE or NEGATIVE, including author name, relative time, and review text.
  5. Returns a single item with fields: `hasReviews`, `businessName`, `ownerEmail`, `ownerName`, `overallRating`, `totalReviews`, `positiveCount`, `negativeCount`, `reviewSummary`, `reviews` (JSON-stringified).
- **Key expressions:** References to config node via `$('2. Set — Config Values').item.json`.
- **Input:** From "3. HTTP — Fetch Google Places Reviews".
- **Output:** Connects to "5. IF — Are There Reviews?".
- **Edge cases:**
  - Google API returning a non-200 status or missing `result` key → code will treat `result` as `{}`, yielding `hasReviews: false`.
  - Reviews missing `author_name`, `text`, or `relative_time_description` → defaults used ("Anonymous", "No written review.", "recently").
  - Reviews with rating 3 are classified as NEGATIVE (≤ 3 threshold).
  - The `reviews` field is JSON-stringified, not an object array, which is intentional for passing as a single string downstream.

#### 5. IF — Are There Reviews?
- **Type:** `n8n-nodes-base.if` (v2.2)
- **Technical Role:** Conditional gate; only allows the workflow to continue if reviews were found.
- **Configuration:**
  - **Combinator:** AND
  - **Condition:** `$json.hasReviews` equals `true` (boolean check).
  - **Type validation:** Loose.
- **Key expressions:** `={{ $json.hasReviews }}`.
- **Input:** From "4. Code — Parse and Classify Reviews".
- **Output (true branch):** Connects to "6. AI Agent — Generate Reply Drafts".
- **Output (false branch):** No connection — workflow terminates silently.
- **Edge cases:** If `hasReviews` is undefined or not a boolean, loose validation may treat it as falsy, stopping the workflow. This is acceptable behavior (fail-safe).

---

### Block 1.3 — AI Reply Generation

**Overview:** A LangChain Agent node receives the formatted review summary along with business context and sends it to GPT-4o-mini. The model writes a reply draft for every review—warm thank-yous for positive reviews and professional apologies with offline-resolution offers for negative reviews. The OpenAI chat model is attached as a sub-node via the `ai_languageModel` connection.

**Nodes Involved:**
- 6. AI Agent — Generate Reply Drafts
- 7. OpenAI — GPT-4o-mini Model

**Node Details:**

#### 6. AI Agent — Generate Reply Drafts
- **Type:** `@n8n/n8n-nodes-langchain.agent` (v1.7)
- **Technical Role:** LangChain Agent that orchestrates the LLM call with a defined prompt.
- **Configuration:**
  - **Prompt Type:** `define` (inline system/user prompt, not from chat trigger).
  - **Text (expression):** Full prompt template including:
    - Role: "professional reputation manager"
    - Business context: `{{ $json.businessName }}`, `{{ $json.overallRating }}`, `{{ $json.ownerName }}`
    - Review summary: `{{ $json.reviewSummary }}`
    - Positive review guidelines: thank by name, mention specific detail, under 60 words
    - Negative review guidelines: apologise, acknowledge concern, offer offline resolution, under 70 words, never defensive
    - Output format: numbered blocks with "REVIEW [n] — [POSITIVE/NEGATIVE] — [author]" / "REPLY DRAFT:" / "---"
    - Rules: plain text only (no markdown/asterisks), sign off with "Best regards, {{ $json.ownerName }}"
- **Key expressions:** `$json.businessName`, `$json.overallRating`, `$json.ownerName`, `$json.reviewSummary`.
- **Input (main):** From "5. IF — Are There Reviews?" (true branch).
- **Input (ai_languageModel):** From "7. OpenAI — GPT-4o-mini Model".
- **Output:** Connects to "8. Set — Prepare Email Fields"; output is accessible via `$json.output` (Agent's text response).
- **Edge cases:**
  - OpenAI credential not connected → execution error.
  - Token limit hit (maxTokens 1000) → reply may be truncated; for many reviews, this limit could be insufficient.
  - Model returning markdown despite instructions → minor formatting issue in the email body.
  - Empty or malformed `reviewSummary` → model may hallucinate reviews; the IF gate should prevent this.

#### 7. OpenAI — GPT-4o-mini Model
- **Type:** `@n8n/n8n-nodes-langchain.lmChatOpenAi` (v1.2)
- **Technical Role:** Chat model sub-node providing LLM capabilities to the Agent.
- **Configuration:**
  - **Model:** `gpt-4o-mini` (selected from list).
  - **Options:**
    - `maxTokens`: 1000
    - `temperature`: 0.7
- **Credential:** Requires an OpenAI API credential configured in n8n.
- **Key expressions:** None.
- **Input:** None (sub-node connected via `ai_languageModel`).
- **Output (ai_languageModel):** Connects to "6. AI Agent — Generate Reply Drafts".
- **Edge cases:**
  - Invalid/expired OpenAI API key → 401 error, workflow fails at the Agent node.
  - Rate limits on OpenAI side → 429 error.
  - `gpt-4o-mini` model not available on the user's OpenAI plan → model not found error.
  - With temperature 0.7, output varies between runs; not deterministic.

---

### Block 1.4 — Email Delivery

**Overview:** A Set node assembles all email fields—recipient address, subject line with review counts and rating, and body text containing the AI-generated drafts plus posting instructions. A Gmail node then sends the digest email to the business owner.

**Nodes Involved:**
- 8. Set — Prepare Email Fields
- 9. Gmail — Send Review Report

**Node Details:**

#### 8. Set — Prepare Email Fields
- **Type:** `n8n-nodes-base.set` (v3.4)
- **Technical Role:** Maps and flattens data from the Code node and Agent output into email-ready fields.
- **Configuration — Assignments:**

| Name | Type | Value Expression | Source |
|---|---|---|---|
| `toEmail` | string | `={{ $('4. Code — Parse and Classify Reviews').item.json.ownerEmail }}` | Config → Code passthrough |
| `emailSubject` | string | `=New Google Reviews Alert — {{ …businessName }} — {{ …totalReviews }} review(s) found — Overall Rating: {{ …overallRating }} stars` | Code node |
| `emailBody` | string | `={{ $json.output \|\| 'AI could not generate reply drafts. Please check OpenAI credentials.' }}` | Agent output with fallback |
| `positiveCount` | number | `={{ $('4. Code — Parse and Classify Reviews').item.json.positiveCount }}` | Code node |
| `negativeCount` | number | `={{ $('4. Code — Parse and Classify Reviews').item.json.negativeCount }}` | Code node |
| `businessName` | string | `={{ $('4. Code — Parse and Classify Reviews').item.json.businessName }}` | Code node |

- **Key expressions:** All reference `$('4. Code — Parse and Classify Reviews').item.json` to pull from the parsed review data, and `$json.output` for the Agent's text.
- **Input:** From "6. AI Agent — Generate Reply Drafts".
- **Output:** Connects to "9. Gmail — Send Review Report".
- **Edge cases:**
  - If the Agent output is `undefined` or empty, `emailBody` falls back to an error message string—good defensive design.
  - If the Code node's output structure changes (e.g., field renamed), all expression references break.

#### 9. Gmail — Send Review Report
- **Type:** `n8n-nodes-base.gmail` (v2.1)
- **Technical Role:** Sends the daily digest email via Gmail.
- **Configuration:**
  - **To (expression):** `={{ $json.toEmail }}`
  - **Subject (expression):** `={{ $json.emailSubject }}`
  - **Message (expression):** Formatted body including:
    - Greeting: "Hi [businessName] Team"
    - Summary section with positive/negative counts
    - "REPLY DRAFTS" section with full AI output (`$json.emailBody`)
    - Step-by-step instructions for posting replies on Google Business Profile (link to https://business.google.com)
    - Footer: "This report was auto-generated by n8n + GPT-4o-mini"
  - **Options:** `appendAttribution` set to `false` (no n8n branding appended).
- **Credential:** Requires a Gmail OAuth2 credential configured in n8n.
- **Key expressions:** `$json.toEmail`, `$json.emailSubject`, `$json.emailBody`, `$json.positiveCount`, `$json.negativeCount`, `$json.businessName`.
- **Input:** From "8. Set — Prepare Email Fields".
- **Output:** None (terminal node).
- **Edge cases:**
  - Gmail OAuth2 token expired → 401 error; token must be refreshed in n8n credentials.
  - Recipient email invalid → bounce/delivery failure.
  - Gmail daily sending quota exceeded → 429 or 403 error.
  - Very large email body (many reviews) could approach Gmail's message size limit (~25 MB, unlikely for text-only).

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Overview | n8n-nodes-base.stickyNote | Visual overview of the entire workflow | — | — | ## Google Reviews Monitor — Auto Reply Drafts via GPT-4o-mini + Gmail. For local business owners, agencies, and reputation managers who want to automatically monitor Google reviews every day and receive AI-written reply drafts by email. The workflow runs daily at 9AM, fetches the latest reviews from Google Places API, classifies them as positive or negative, and uses GPT-4o-mini to write a personalised reply for each one. Gmail delivers a clean digest with all drafts ready to copy and paste. / How it works: 1. Schedule triggers daily at 9AM; 2. Set stores config; 3. HTTP fetches reviews; 4. Code parses & classifies; 5. IF checks for reviews; 6. AI Agent generates replies; 8. Set prepares email; 9. Gmail sends report. / Set up steps: 1. In Set—Config Values replace all five placeholders; 2. In OpenAI node connect credential; 3. In Gmail node connect OAuth2; 4. Activate workflow. |
| Section — Schedule and Config | n8n-nodes-base.stickyNote | Section label for schedule and config block | — | — | ## Schedule and Config — Workflow triggers every day at 9AM. All config values — Place ID, API key, business name, recipient email, and owner name — are set here once and used throughout. |
| Section — Review Fetch and Classification | n8n-nodes-base.stickyNote | Section label for fetch/classify block | — | — | ## Review Fetch and Classification — Fetches the latest reviews from Google Places API. Parses the response, separates positive (4–5 stars) from negative (1–3 stars), and checks if any reviews exist before continuing. |
| Section — AI Reply Generation | n8n-nodes-base.stickyNote | Section label for AI generation block | — | — | ## AI Reply Generation — GPT-4o-mini reads all review texts and writes a personalised reply draft for each one. Positive reviews get a warm thank-you. Negative reviews get a professional apology with an offline resolution offer. |
| Section — Email Delivery | n8n-nodes-base.stickyNote | Section label for email delivery block | — | — | ## Email Delivery — Assembles the email subject, recipient, and body with all reply drafts. Gmail sends the daily digest to the business owner. |
| Note — Edit Config Before Activating | n8n-nodes-base.stickyNote | Warning to edit config node | — | — | ## ⚠️ Edit This Node Before Activating — This is the only node you need to change. Replace all five values: Place ID (find it at developers.google.com/maps/documentation/places/web-service/place-id), Google Places API key (from Google Cloud Console with Places API enabled), business name, owner email, and owner name. |
| 1. Schedule — Every Day 9AM | n8n-nodes-base.scheduleTrigger | Daily cron trigger at 09:00 | — | 2. Set — Config Values | ## Schedule and Config — Workflow triggers every day at 9AM. All config values — Place ID, API key, business name, recipient email, and owner name — are set here once and used throughout. |
| 2. Set — Config Values | n8n-nodes-base.set | Central configuration store for Place ID, API key, business name, owner email, owner name | 1. Schedule — Every Day 9AM | 3. HTTP — Fetch Google Places Reviews | ## Schedule and Config — Workflow triggers every day at 9AM. All config values — Place ID, API key, business name, recipient email, and owner name — are set here once and used throughout. / ## ⚠️ Edit This Node Before Activating — This is the only node you need to change. Replace all five values: Place ID (find it at developers.google.com/maps/documentation/places/web-service/place-id), Google Places API key (from Google Cloud Console with Places API enabled), business name, owner email, and owner name. |
| 3. HTTP — Fetch Google Places Reviews | n8n-nodes-base.httpRequest | GET request to Google Places Details API for reviews | 2. Set — Config Values | 4. Code — Parse and Classify Reviews | ## Review Fetch and Classification — Fetches the latest reviews from Google Places API. Parses the response, separates positive (4–5 stars) from negative (1–3 stars), and checks if any reviews exist before continuing. |
| 4. Code — Parse and Classify Reviews | n8n-nodes-base.code | Parses API response, classifies reviews by sentiment, builds text summary | 3. HTTP — Fetch Google Places Reviews | 5. IF — Are There Reviews? | ## Review Fetch and Classification — Fetches the latest reviews from Google Places API. Parses the response, separates positive (4–5 stars) from negative (1–3 stars), and checks if any reviews exist before continuing. |
| 5. IF — Are There Reviews? | n8n-nodes-base.if | Conditional gate; continues only if reviews exist | 4. Code — Parse and Classify Reviews | 6. AI Agent — Generate Reply Drafts (true branch) | ## Review Fetch and Classification — Fetches the latest reviews from Google Places API. Parses the response, separates positive (4–5 stars) from negative (1–3 stars), and checks if any reviews exist before continuing. |
| 6. AI Agent — Generate Reply Drafts | @n8n/n8n-nodes-langchain.agent | LangChain Agent that generates personalised reply drafts via GPT-4o-mini | 5. IF — Are There Reviews? (main), 7. OpenAI — GPT-4o-mini Model (ai_languageModel) | 8. Set — Prepare Email Fields | ## AI Reply Generation — GPT-4o-mini reads all review texts and writes a personalised reply draft for each one. Positive reviews get a warm thank-you. Negative reviews get a professional apology with an offline resolution offer. |
| 7. OpenAI — GPT-4o-mini Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | Chat model sub-node providing LLM inference to the Agent | — | 6. AI Agent — Generate Reply Drafts (ai_languageModel) | ## AI Reply Generation — GPT-4o-mini reads all review texts and writes a personalised reply draft for each one. Positive reviews get a warm thank-you. Negative reviews get a professional apology with an offline resolution offer. |
| 8. Set — Prepare Email Fields | n8n-nodes-base.set | Maps parsed review data and Agent output into email fields | 6. AI Agent — Generate Reply Drafts | 9. Gmail — Send Review Report | ## Email Delivery — Assembles the email subject, recipient, and body with all reply drafts. Gmail sends the daily digest to the business owner. |
| 9. Gmail — Send Review Report | n8n-nodes-base.gmail | Sends the daily review digest email via Gmail | 8. Set — Prepare Email Fields | — | ## Email Delivery — Assembles the email subject, recipient, and body with all reply drafts. Gmail sends the daily digest to the business owner. |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in your n8n instance.

2. **Add a Schedule Trigger node.**
   - Type: `Schedule Trigger`
   - Configuration: Set trigger type to "Cron Expression" with value `0 9 * * *`.
   - This fires the workflow daily at 09:00 server time.

3. **Add a Set node** and connect it after the Schedule Trigger.
   - Type: `Set` (v3.4)
   - Name it "2. Set — Config Values".
   - Add five string assignments:
     - `placeId` → `YOUR_GOOGLE_PLACE_ID`
     - `apiKey` → `YOUR_GOOGLE_PLACES_API_KEY`
     - `businessName` → `YOUR BUSINESS NAME`
     - `ownerEmail` → `user@example.com`
     - `ownerName` → `YOUR NAME`
   - These are placeholders; replace before activating.

4. **Add an HTTP Request node** and connect it after the Set node.
   - Type: `HTTP Request` (v4.2)
   - Name it "3. HTTP — Fetch Google Places Reviews".
   - Method: GET
   - URL (expression): `https://maps.googleapis.com/maps/api/place/details/json?place_id={{ $json.placeId }}&fields=name,rating,reviews&key={{ $json.apiKey }}`
   - No authentication header needed; the API key is in the query string.

5. **Add a Code node** and connect it after the HTTP Request node.
   - Type: `Code` (v2)
   - Name it "4. Code — Parse and Classify Reviews".
   - Paste the following JavaScript (same logic as original):
     ```javascript
     const data = $input.first().json;
     const config = $('2. Set — Config Values').item.json;

     const result = data.result || {};
     const reviews = result.reviews || [];
     const overallRating = result.rating || 'N/A';

     if (reviews.length === 0) {
       return [{
         json: {
           hasReviews: false,
           message: 'No reviews found for this business.',
           businessName: config.businessName,
           ownerEmail: config.ownerEmail,
           ownerName: config.ownerName,
           overallRating: overallRating,
           reviews: []
         }
       }];
     }

     const positiveReviews = reviews.filter(r => r.rating >= 4);
     const negativeReviews = reviews.filter(r => r.rating <= 3);

     let reviewSummary = '';
     reviews.forEach((r, i) => {
       const stars = r.rating || 0;
       const author = r.author_name || 'Anonymous';
       const text = r.text || 'No written review.';
       const time = r.relative_time_description || 'recently';
       const sentiment = stars >= 4 ? 'POSITIVE' : 'NEGATIVE';
       reviewSummary += `Review ${i + 1} [${sentiment} - ${stars} stars]\nAuthor: ${author} (${time})\nReview: ${text}\n\n`;
     });

     return [{
       json: {
         hasReviews: true,
         businessName: config.businessName,
         ownerEmail: config.ownerEmail,
         ownerName: config.ownerName,
         overallRating: overallRating,
         totalReviews: reviews.length,
         positiveCount: positiveReviews.length,
         negativeCount: negativeReviews.length,
         reviewSummary: reviewSummary,
         reviews: JSON.stringify(reviews)
       }
     }];
     ```
   - The code references the Set node by name, so ensure the Set node's name matches exactly.

6. **Add an IF node** and connect it after the Code node.
   - Type: `IF` (v2.2)
   - Name it "5. IF — Are There Reviews?".
   - Condition: `$json.hasReviews` equals `true` (boolean, loose validation, AND combinator).
   - Connect the **true** branch to the next node (step 7). Leave the **false** branch unconnected (workflow ends).

7. **Add an AI Agent node** and connect it to the true output of the IF node.
   - Type: `AI Agent` (`@n8n/n8n-nodes-langchain.agent`, v1.7)
   - Name it "6. AI Agent — Generate Reply Drafts".
   - Prompt Type: `define`.
   - Text (expression):
     ```
     You are a professional reputation manager for a local business.

     Your job is to write personalised reply drafts for Google reviews.

     BUSINESS NAME: {{ $json.businessName }}
     OVERALL RATING: {{ $json.overallRating }} stars
     OWNER NAME: {{ $json.ownerName }}

     REVIEWS TO RESPOND TO:
     {{ $json.reviewSummary }}

     For EACH review above write a reply draft.

     For POSITIVE reviews (4-5 stars):
     - Start by thanking the reviewer by name
     - Mention one specific thing they said in their review
     - Express genuine appreciation
     - Invite them to visit or contact again
     - Keep it warm, friendly, and under 60 words

     For NEGATIVE reviews (1-3 stars):
     - Start by apologising sincerely
     - Acknowledge their concern without making excuses
     - Offer to resolve the issue offline (ask them to contact directly)
     - Keep it professional, calm, and under 70 words
     - Never argue or be defensive

     OUTPUT FORMAT:
     For each review write exactly this structure:

     REVIEW [number] — [POSITIVE or NEGATIVE] — [author name]
     REPLY DRAFT:
     [your reply text here]
     ---

     RULES:
     - Plain text only. No markdown. No asterisks.
     - End every reply with: Best regards, {{ $json.ownerName }}
     - Write all replies one after another separated by ---
     ```
   - No tools connected; only the language model sub-node.

8. **Add an OpenAI Chat Model sub-node** and connect it to the AI Agent.
   - Type: `OpenAI Chat Model` (`@n8n/n8n-nodes-langchain.lmChatOpenAi`, v1.2)
   - Name it "7. OpenAI — GPT-4o-mini Model".
   - Model: select `gpt-4o-mini` from the list.
   - Options: `maxTokens` = 1000, `temperature` = 0.7.
   - **Credential:** Create or select an OpenAI API credential in n8n (requires a valid API key with access to `gpt-4o-mini`).
   - Connect the output to the AI Agent's `ai_languageModel` input.

9. **Add a Set node** and connect it after the AI Agent.
   - Type: `Set` (v3.4)
   - Name it "8. Set — Prepare Email Fields".
   - Add six assignments:
     - `toEmail` (string): `={{ $('4. Code — Parse and Classify Reviews').item.json.ownerEmail }}`
     - `emailSubject` (string): `=New Google Reviews Alert — {{ $('4. Code — Parse and Classify Reviews').item.json.businessName }} — {{ $('4. Code — Parse and Classify Reviews').item.json.totalReviews }} review(s) found — Overall Rating: {{ $('4. Code — Parse and Classify Reviews').item.json.overallRating }} stars`
     - `emailBody` (string): `={{ $json.output || 'AI could not generate reply drafts. Please check OpenAI credentials.' }}`
     - `positiveCount` (number): `={{ $('4. Code — Parse and Classify Reviews').item.json.positiveCount }}`
     - `negativeCount` (number): `={{ $('4. Code — Parse and Classify Reviews').item.json.negativeCount }}`
     - `businessName` (string): `={{ $('4. Code — Parse and Classify Reviews').item.json.businessName }}`

10. **Add a Gmail node** and connect it after the Set node.
    - Type: `Gmail` (v2.1)
    - Name it "9. Gmail — Send Review Report".
    - **Credential:** Create or select a Gmail OAuth2 credential in n8n (authorize n8n to send emails on your behalf).
    - Parameters:
      - **Send To** (expression): `={{ $json.toEmail }}`
      - **Subject** (expression): `={{ $json.emailSubject }}`
      - **Message** (expression):
        ```
        Hi {{ $json.businessName }} Team,

        Your daily Google Reviews report is ready.

        SUMMARY
        Positive Reviews (4-5 stars): {{ $json.positiveCount }}
        Negative Reviews (1-3 stars): {{ $json.negativeCount }}

        REPLY DRAFTS — Copy and paste these directly to Google Business Profile:

        {{ $json.emailBody }}

        ---
        To post replies:
        1. Go to Google Business Profile: https://business.google.com
        2. Click Reviews
        3. Find the review and paste the draft
        4. Edit if needed and click Reply

        This report was auto-generated by n8n + GPT-4o-mini
        ```
      - **Options:** `appendAttribution` = false.

11. **Replace placeholder values** in the "2. Set — Config Values" node:
    - `placeId`: Your Google Place ID (find at https://developers.google.com/maps/documentation/places/web-service/place-id).
    - `apiKey`: Your Google Places API key (from Google Cloud Console; ensure the Places API is enabled).
    - `businessName`: Your actual business name.
    - `ownerEmail`: The email address that should receive the daily digest.
    - `ownerName`: The name to use as sign-off in AI replies.

12. **Verify credentials:**
    - OpenAI API credential connected to node 7 with a key that has access to `gpt-4o-mini`.
    - Gmail OAuth2 credential connected to node 9 with permission to send emails.

13. **Activate the workflow.** Once activated, it will execute automatically every day at 9:00 AM (server timezone). You can also click "Execute Workflow" manually to test.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Google Place ID lookup tool — required to find your business Place ID | https://developers.google.com/maps/documentation/places/web-service/place-id |
| Google Cloud Console — enable the Places API and generate an API key | https://console.cloud.google.com |
| Google Business Profile — where to post the reply drafts | https://business.google.com |
| The Google Places Details API returns up to 5 most recent reviews per call by default; businesses with higher review volumes may need additional handling or the Google Business Profile API for full access | Google Places API documentation |
| GPT-4o-mini model availability depends on your OpenAI plan and region; verify at OpenAI's model page | https://platform.openai.com/docs/models |
| Gmail OAuth2 tokens in n8n may expire and require re-authorization; monitor credential health in n8n's credential manager | n8n credential management |
| The workflow has no error handling branch on the IF node's false output — if no reviews are found, execution ends silently with no notification. Consider adding a "No reviews today" email or Slack message on the false branch if you want explicit confirmation | Enhancement suggestion |
| maxTokens is set to 1000 on GPT-4o-mini; for businesses with many reviews (e.g., 5 detailed reviews), the total reply text may approach or exceed this limit, resulting in truncated output. Increase to 2000–3000 if truncation is observed | Configuration tuning |
| Temperature of 0.7 introduces variability in generated replies; for more consistent/deterministic tone, reduce to 0.3–0.5 | Configuration tuning |
| n8n server timezone determines the actual local time of the 9 AM cron trigger; verify your n8n instance timezone in environment settings | Deployment consideration |