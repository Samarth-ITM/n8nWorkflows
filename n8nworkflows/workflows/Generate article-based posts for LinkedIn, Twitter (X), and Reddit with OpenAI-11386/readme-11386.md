Generate article-based posts for LinkedIn, Twitter (X), and Reddit with OpenAI

https://n8nworkflows.xyz/workflows/generate-article-based-posts-for-linkedin--twitter--x---and-reddit-with-openai-11386


# Generate article-based posts for LinkedIn, Twitter (X), and Reddit with OpenAI

## 1. Workflow Overview

**Title:** Generate article-based posts for LinkedIn, Twitter (X), and Reddit with OpenAI  
**Purpose:** Turn a submitted article URL into (1) a LinkedIn post optimized using RAG from a Supabase “viral posts” dataset, (2) a Twitter/X tweet under 280 characters, and (3) a Reddit link post with a selected subreddit flair. The workflow includes **interactive review/regeneration loops** before posting (LinkedIn/Twitter), plus automated Reddit posting.

### 1.1 Input Reception & Validation
User submits an article URL via an authenticated n8n Form Trigger. The workflow validates it is a URL before continuing.

### 1.2 Article Scraping & Content Extraction
Fetches HTML for the URL, then uses a Code node with **Mozilla Readability + JSDOM** to extract clean article text and metadata (title, author, excerpt, image, etc.). If scraping/parsing fails, it returns a “Unable to Process” form.

### 1.3 LinkedIn Post Generation (RAG + Multi-Agent) + Review Loop + Publishing
Uses:
- **Supabase vector store** (viral LinkedIn posts) as a tool
- **OpenAI embeddings** for retrieval
- A strategist agent (RAG analysis) → generator agent (draft) → formatter agent (final)
Then two-step review/regeneration loop, optional image preview, and finally posts to LinkedIn as **Text+Image** or **Text+Link**.

### 1.4 Twitter/X Generation + Review Loop + Tweeting
A Twitter agent generates a tweet with URL under 280 chars (structured output). User can request a regeneration once, then it posts via Twitter node (with an image attachment reference).

### 1.5 Reddit Posting (Subreddit selection → flair retrieval → AI flair choice → submit)
User selects a subreddit from a dropdown, the workflow fetches available flairs, aggregates them, asks an OpenAI node to pick one, and submits a link post.

---

## 2. Block-by-Block Analysis

### Block 1 — Input Reception & URL Validation
**Overview:** Collects the article link securely and ensures it is a valid URL before scraping.  
**Nodes involved:** `On Article Submission`, `If`

#### Node: On Article Submission
- **Type / role:** `Form Trigger` — Entry point (web form + webhook).
- **Config choices:**
  - Form path: `article-post`
  - Field: `Article Link` (textarea, required)
  - **Basic Auth** enabled to restrict access
  - Response mode: `lastNode` (user sees output of whichever node ends the executed branch)
  - Ignore bots enabled
- **Key variables:** `$json['Article Link']`
- **Connections:** Outputs to `If`.
- **Failure/edge cases:**
  - Auth failure (401) if Basic Auth incorrect
  - Users submit non-URL content; handled by next node

#### Node: If
- **Type / role:** `If` — Validates URL format.
- **Condition:** `{{ $json['Article Link'].isUrl() }} == true`
- **Connections:** If true → `Scrape Article`. (No explicit false path wired; invalid URLs effectively end without a user-facing completion unless n8n form behavior returns last executed node from prior steps.)
- **Edge cases:**
  - `isUrl()` depends on n8n expression helpers; malformed strings return false.
  - Missing explicit “false” branch is a UX gap (recommended: connect false to a “Please enter a valid URL” completion form).

---

### Block 2 — Article Scraping & Readability Extraction
**Overview:** Downloads the raw HTML and extracts the main article content + metadata, including a best-effort featured image URL.  
**Nodes involved:** `Scrape Article`, `Extract Details`, `⚠️Unable to Process Article`, `Sticky Note` (comment)

#### Node: Scrape Article
- **Type / role:** `HTTP Request` — Fetches the article HTML.
- **Config choices:**
  - URL: `{{ $json['Article Link'] }}`
  - `retryOnFail: true`
  - `onError: continueErrorOutput` (important: errors go to a second output)
- **Outputs:**
  - Output 0 (success) → `Extract Details`
  - Output 1 (error) → `⚠️Unable to Process Article`
- **Failure/edge cases:**
  - 403/429 from bot protection, paywalls, Cloudflare
  - Non-HTML responses
  - Timeout/large pages
  - Because “continue on error” is enabled, downstream nodes must not assume `data` exists.

#### Node: Extract Details
- **Type / role:** `Code` — Parses HTML and extracts article fields.
- **Config choices (interpreted):**
  - Uses `@mozilla/readability` to parse a clean article representation.
  - Uses `jsdom` to construct DOM and read metadata tags.
  - Determines image URL via priority:
    1. `meta[property="og:image"]`
    2. `meta[name="twitter:image"]`
    3. First `<img>` inside Readability `article.content`
  - Resolves relative image URLs against the article URL.
- **Expected input JSON:** from `Scrape Article` with fields like `data` (HTML) and `url` (final URL). The code assumes:
  - `const html = inputData.data;`
  - `const url = inputData.url`
- **Output JSON fields (on success):**
  - `title`, `author`, `contentText`, `wordCount`, `siteName`, `excerpt`, `imageUrl`
- **Output JSON fields (on failure):**
  - `error`, plus diagnostics like `stack`, `inputKeys`, or Readability failure reason.
- **Connections:** Success output → `LinkedIn Post Strategist`, `Twitter Agent`, `Reddit Form` (fan-out).
- **Failure/edge cases:**
  - If `Scrape Article` returns binary or different field names, `inputData.data` may not be the HTML string.
  - Some pages provide minimal HTML or require JS rendering; Readability returns null.
  - Extremely large HTML can increase memory usage in Code node.
- **Version-specific notes:** Code node v2 uses Node.js runtime with `require()`; installed packages must be available in the n8n environment (self-host often needs these dependencies available).

#### Node: ⚠️Unable to Process Article
- **Type / role:** `Form (completion)` — User-facing failure page.
- **Config:** Title “⚠️Unable to Process Article”, message “Please enter different Article”.
- **Connection:** Terminal.

#### Sticky Note (comment coverage)
- **Content:** “Extract Article Content - Title, Article Content, Author, Excerpt, Website Name, Article Image”
- **Applies to:** The extraction block nodes.

---

### Block 3 — LinkedIn RAG + Generation + Formatting
**Overview:** Uses a Supabase vector store of viral LinkedIn posts to derive patterns, then generates and formats a LinkedIn post that includes the article URL.  
**Nodes involved:** `Embedding`, `LinkedIn Post Vector Store`, `Structured Output Parser`, `5 mini`, `4.1`, `5`, `LinkedIn Post Strategist`, `LinkedIn Post Generator`, `LinkedIn Post Formatter`, `Sticky Note10`

#### Node: Embedding
- **Type / role:** `OpenAI Embeddings` — Supplies embeddings for vector retrieval.
- **Credentials:** OpenAI API.
- **Connections:** `ai_embedding` → `LinkedIn Post Vector Store`.
- **Failure/edge cases:** API auth errors, model availability, rate limits.

#### Node: LinkedIn Post Vector Store
- **Type / role:** `Supabase Vector Store` — Retrieval tool for the strategist agent.
- **Mode:** “retrieve-as-tool” (the agent can call it as a tool).
- **Table:** `linkedin_post`
- **Credentials:** Supabase API.
- **Connections:** Tool output wired to `LinkedIn Post Strategist` as `ai_tool`.
- **Failure/edge cases:**
  - Supabase auth / row-level security blocking reads
  - Table not existing, missing vector column/index depending on how it was created

#### Node: Structured Output Parser
- **Type / role:** `Structured Output Parser` — Enforces JSON output schema.
- **Schema example:** `{ "Post Content": "Post Content" }`
- **autoFix:** enabled (tries to repair invalid JSON).
- **Connections:** Output parser is attached to `LinkedIn Post Strategist` (`ai_outputParser`).
- **Edge cases:** If the agent produces content incompatible with schema, autoFix may still fail or distort content.

#### Nodes: 5 mini / 4.1 / 5 (LinkedIn model providers)
- **Type / role:** `LM Chat OpenAI` — Provide different model options to LangChain agents/parsers.
- **Models:**
  - `5 mini`: `gpt-5-mini` (wired to structured parser)
  - `4.1`: `gpt-4.1` (wired to multiple LinkedIn agents)
  - `5`: `gpt-5` (wired to multiple LinkedIn agents)
- **Connections:** They supply `ai_languageModel` inputs to the agents/parsers.
- **Edge cases:** Model access not enabled for the API key; higher latency/cost for larger models.

#### Node: LinkedIn Post Strategist
- **Type / role:** `LangChain Agent` — Analyzes article and viral post patterns using the vector tool.
- **Input text:** Uses extracted fields:
  - `{{ $json.title }}`, `{{ $json.author }}`, `{{ $json.contentText }}`, `{{ $json.siteName }}`, `{{ $json.excerpt }}`
- **System message:** Long “LinkedIn Viral Content Analyzer” prompt, including:
  - Tool usage instructions
  - Formatting rules: no em dashes, Sans Serif Bold Unicode for “bold sections”
- **Tools:** `LinkedIn Post Vector Store` available.
- **Output parser:** `Structured Output Parser` attached (but note: the strategist prompt describes analysis, while the schema demands `"Post Content"`, which can be mismatched unless the agent is actually expected to output JSON. This is a potential design inconsistency.)
- **Connections:** Main output → `LinkedIn Post Generator`.
- **Edge cases:**
  - Retrieval tool may return irrelevant posts if embeddings/table are not aligned.
  - If output parser forces strategist into `"Post Content"` while it should output “strategy”, downstream generator may not receive intended insights.

#### Node: LinkedIn Post Generator
- **Type / role:** `LangChain Agent` — Generates a LinkedIn post draft.
- **Input text:**
  - `Content Text: {{ $('Extract Details').item.json.contentText }}`
  - `Article URL: {{ $('On Article Submission').item.json['Article Link'] }}`
- **System message:** “Generate linkedin post… Must add Article URL… {{ $json.output['Post Content'] }}”
  - It attempts to inject strategist output via `$json.output['Post Content']` (depends on strategist producing that field).
- **Connections:** → `LinkedIn Post Formatter`.
- **Edge cases:** If strategist output isn’t shaped as expected, the generator prompt may include `undefined`.

#### Node: LinkedIn Post Formatter
- **Type / role:** `LangChain Agent` — Cleans output to “post only”.
- **Input text:** `{{ $json.output }}` (expects generator’s `output` field).
- **System rules:**
  - Remove any extra explanation, keep article URL
  - Use Sans Serif Bold Unicode for emphasis
  - No em dashes, no markdown
- **Connections:** → `Post Review Form`
- **Edge cases:** Formatter may remove required URL if the model misunderstands; consider adding explicit “ensure URL present” check.

#### Sticky Note10
- **Content:** “LinkedIn”
- **Applies to:** LinkedIn generation/publishing area.

---

### Block 4 — LinkedIn Review Loop + Image Handling + Publish
**Overview:** Shows generated LinkedIn post to the user, optionally regenerates once (with a second review), then previews the extracted image and posts either with image or link preview.  
**Nodes involved:** `Post Review Form`, `Post Review Condition`, `Re-generate LinkedIn Post`, `Post 2 Review Form`, `Post 2 Review Conditions`, `Download Image for Form`, `Image Preview Form`, `Image Preview Conditions`, `Download Image for LinkedIn Post`, `Text + Image`, `Text + Link`, `The End`

#### Node: Post Review Form
- **Type / role:** `Form` — Displays the formatted post and collects feedback.
- **HTML display:** `{{ $json.output.replaceAll("\n","<br>") }}`
- **Fields:**
  - Radio: “Want to re-generate post content?” (Yes/No)
  - Textarea: “Describe Post Changes...”
- **Connections:** → `Post Review Condition`
- **Edge cases:** If `$json.output` is missing, the HTML rendering expression fails.

#### Node: Post Review Condition
- **Type / role:** `If` — Branches based on radio selection.
- **Condition:** If `Want to re-generate post content? == "Yes"`
- **Outputs:**
  - True → `Re-generate LinkedIn Post`
  - False → `Download Image for Form`
- **Edge cases:** If user leaves fields empty (though radio required), string mismatch.

#### Node: Re-generate LinkedIn Post
- **Type / role:** `LangChain Agent` — Rewrites post using user feedback.
- **Prompt references:**
  - Original: `{{ $('LinkedIn Post Formatter').item.json.output }}`
  - Feedback: `{{ $json['Describe Post Changes...'] }}`
- **Rules:** post only, bold unicode, no em dashes, no markdown.
- **Connections:** → `Post 2 Review Form`

#### Node: Post 2 Review Form
- **Type / role:** `Form` — Second approval step.
- **Radio:** “Want to re-generate post content again?” (Yes/No)
- **Connections:** → `Post 2 Review Conditions`
- **Potential issue:** Uses same `webhookId` as other forms in workflow; in practice, each Form node should have a distinct webhookId/path to avoid collisions (n8n normally enforces uniqueness).

#### Node: Post 2 Review Conditions
- **Type / role:** `If`
- **Condition:** `Want to re-generate post content again? == "No"`
- **Outputs (as wired):**
  - True → `Download Image for Form`
  - False → `The End`
- **Meaning:** Only allows **one** regeneration cycle. If user wants another, it ends with an error completion message.

#### Node: The End
- **Type / role:** `Form (completion)` — Shows message that further regeneration is not supported.
- **Text:** “Please submit article Again… I can't able to re-generate post again...”

#### Node: Download Image for Form
- **Type / role:** `HTTP Request` — Downloads the extracted image (for preview step).
- **URL:** `{{ $('Extract Details').item.json.imageUrl }}`
- **Connections:** → `Image Preview Form`
- **Edge cases:** `imageUrl` null/empty leads to request error; there is no explicit guard.

#### Node: Image Preview Form
- **Type / role:** `Form` — Shows a link to the article image and asks whether to use it.
- **Radio:** “Want to continue with above Image?” options:
  - `Yes`
  - `Continue without Image`
- **Connections:** → `Image Preview Conditions`

#### Node: Image Preview Conditions
- **Type / role:** `If`
- **Condition:** `Want to continue with above Image? == "Yes"`
- **Outputs:**
  - True → `Download Image for LinkedIn Post`
  - False → `Text + Link`

#### Node: Download Image for LinkedIn Post
- **Type / role:** `HTTP Request` — Fetches the image binary for LinkedIn upload.
- **URL:** `{{ $('Extract Details').item.json.imageUrl }}`
- **Connections:** → `Text + Image`
- **Edge cases:** Same as image download above; LinkedIn may reject unsupported formats/large files.

#### Node: Text + Image
- **Type / role:** `LinkedIn` — Posts an IMAGE share.
- **Text:** `{{ $('LinkedIn Post Formatter').item.json.output ?? $('Re-generate LinkedIn Post').item.json.output }}`
  - Uses nullish-coalescing to prefer first version, otherwise regenerated.
- **Person ID:** `A_SsE0JFbA` (must be replaced with your LinkedIn member URN/ID as required by node)
- **shareMediaCategory:** `IMAGE`
- **Credentials:** LinkedIn OAuth2
- **Edge cases:** Permission scopes missing (posting), invalid person id, media upload errors.

#### Node: Text + Link
- **Type / role:** `LinkedIn` — Posts an ARTICLE share (link preview).
- **Text:** same as above
- **Additional fields:**
  - `title`: extracted article title
  - `originalUrl`: submitted article link
- **shareMediaCategory:** `ARTICLE`
- **Edge cases:** Link preview failures, LinkedIn API restrictions.

---

### Block 5 — Twitter/X Generation + Review Loop + Tweet
**Overview:** Generates a tweet (structured) with article link under 280 chars, offers a review/regeneration step, then posts through Twitter node with an attachment reference.  
**Nodes involved:** `Twitter Agent`, `Structured Output Parser X`, `5 minii`, `4.i`, `5.`, `Tweet Review Form`, `Tweeet Review Conditions`, `Re-generate Tweet Agent`, `Tweet 2 Review Form`, `Tweet 2 Review Conditions`, `Tweet`, `Re-generated Tweet`, `Sticky Note4`

#### Node: Twitter Agent
- **Type / role:** `LangChain Agent` — Creates tweet content.
- **Input text:** article fields plus URL.
- **System rules:**
  - Must be under 280 chars including spaces
  - Must include article link
- **Output parser:** `Structured Output Parser X` (expects JSON with `"Post Content"`).
- **Connections:** Main output → `Tweet Review Form`
- **Edge cases:** Character counting by model can be inaccurate; consider post-validation in Code node.

#### Node: Structured Output Parser X
- **Type / role:** Structured output enforcement for Twitter agent.
- **Schema:** `{ "Post Content": "Post Content" }`
- **autoFix:** enabled
- **Connections:** attached to `Twitter Agent` as `ai_outputParser`

#### Nodes: 5 minii / 4.i / 5. (Twitter model providers)
- **Type / role:** `LM Chat OpenAI`
- **Models:** `gpt-5-mini`, `gpt-4.1`, `gpt-5` connected as language models to Twitter agent / regenerate agent.
- **Edge cases:** access/latency/cost.

#### Node: Tweet Review Form
- **Type / role:** `Form` — Displays tweet and asks if regenerate.
- **HTML display:** `{{ $json.output['Post Content'].replaceAll("\n","<br>") }}`
- **Radio:** “Want to re-generate Tweet?” Yes/No
- **Connections:** → `Tweeet Review Conditions`

#### Node: Tweeet Review Conditions
- **Type / role:** `If`
- **Condition:** `Want to re-generate Tweet? != "Yes"`
- **Outputs (as wired):**
  - True (not equals Yes) → `Tweet`
  - False (equals Yes) → `Re-generate Tweet Agent`

#### Node: Tweet
- **Type / role:** `Twitter` — Posts the tweet.
- **Text:** `{{ $('Twitter Agent').item.json.output['Post Content'] }}`
- **Attachments field:** `{{ $json.data.id }}`
  - This expects an upstream step to provide `data.id` (a media upload result). **No media upload node exists in this workflow**, so this is likely to be undefined and may cause API errors.
- **Credentials:** Twitter OAuth2
- **Edge cases:** Missing media id, invalid scopes, duplicate content restrictions, rate limits.

#### Node: Re-generate Tweet Agent
- **Type / role:** `LangChain Agent` — Rewrite tweet based on feedback.
- **Prompt references:**
  - Original: `{{ $('Twitter Agent').item.json.output['Post Content'] }}`
  - Feedback: `{{ $json['Describe Post Changes...'] }}`
- **Connections:** → `Tweet 2 Review Form`

#### Node: Tweet 2 Review Form
- **Type / role:** `Form` — Second approval step for regenerated tweet.
- **Radio label:** “Want to re-generate Tweet again?”
- **Connections:** → `Tweet 2 Review Conditions`

#### Node: Tweet 2 Review Conditions
- **Type / role:** `If`
- **Condition configured:** checks `{{ $json['Want to re-generate Tweet again??'] }} == "No"`
- **Important issue:** The form field label is “Want to re-generate Tweet again?” (single `?`), but the IF checks “again??” (double `??`). This mismatch likely makes the condition always evaluate as false/undefined.
- **Wiring:** Output index 1 → `Re-generated Tweet`; output index 0 is unused.
- **Net effect:** The regenerated tweet posting path is unreliable and may not run as intended.

#### Node: Re-generated Tweet
- **Type / role:** `Twitter` — Posts the regenerated tweet.
- **Text:** `{{ $('Re-generate Tweet Agent').item.json.output }}`
- **Attachments:** `{{ $json.data.id }}` (same missing-media issue)
- **Credentials:** Twitter OAuth2

#### Sticky Note4
- **Content:** “X (Twitter)”
- **Applies to:** Twitter section nodes.

---

### Block 6 — Reddit Posting (Subreddit + Flair + Submit)
**Overview:** Lets user choose subreddit, retrieves flairs, asks AI to select the best flair for the title, and submits the article as a Reddit link post.  
**Nodes involved:** `Reddit Form`, `Get Flair`, `[Flair]`, `Flair Selector Agent`, `Reddit Post`, `Sticky Note5`

#### Node: Reddit Form
- **Type / role:** `Form` — User chooses subreddit.
- **Dropdown options:** `r/n8n`, `r/mcp`, `r/technews`
- **Connections:** → `Get Flair`

#### Node: Get Flair
- **Type / role:** `HTTP Request` — Calls Reddit API to list flairs for the chosen subreddit.
- **URL:** `https://oauth.reddit.com/r/{{ $json.Subreddit.split("/")[1] }}/api/link_flair_v2`
- **Auth:** OAuth2 (generic credential type)
- **Credentials listed:** Both `oAuth2Api` and `redditOAuth2Api` appear present; node uses generic OAuth2.
- **Connections:** → `[Flair]`
- **Edge cases:** Token expiration, missing scopes, subreddit restrictions, rate limits.

#### Node: [Flair]
- **Type / role:** `Aggregate` — Collects flair texts into a single field.
- **Operation:** aggregates `text` → output field `flair`
- **Connections:** → `Flair Selector Agent`
- **Edge cases:** If Reddit returns empty flairs, `flair` will be empty and AI selection becomes unreliable.

#### Node: Flair Selector Agent
- **Type / role:** `OpenAI (non-agent) node` — Picks one flair from the list.
- **Model:** `gpt-4o-mini`
- **Prompt:** “Subreddit Flairs: {{ $json.flair }}; Title: … Select only one appropriate flair… Output plain text only”
- **Connections:** → `Reddit Post`
- **Edge cases:** Output is plain text but Reddit Post node is not wired to consume it (see below).

#### Node: Reddit Post
- **Type / role:** `HTTP Request` — Submits link to Reddit.
- **URL:** `https://oauth.reddit.com/r/n8n/api/submit` (hardcoded to r/n8n)
- **Method:** POST, send query parameters:
  - `title`: extracted article title
  - `kind`: `link`
  - `flair_id`: hardcoded `c167b438-1bc8-11f0-8814-1e37289161d8`
  - `url`: submitted article link
  - `sr`: **parameter present but has no value set**
- **Auth:** OAuth2
- **Major functional gaps / edge cases:**
  - Subreddit is hardcoded to `/r/n8n/` and does not use the dropdown selection.
  - `sr` query parameter is blank; Reddit expects `sr` to be subreddit name.
  - Flair selection output is not used; `flair_id` is hardcoded, ignoring AI-selected flair.
  - Likely to fail or post incorrectly unless the hardcoded flair id matches the hardcoded subreddit and `sr` is not required (usually it is).

#### Sticky Note5
- **Content:** “Reddit”
- **Applies to:** Reddit section nodes.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| On Article Submission | Form Trigger | Collect article URL (secured) and start workflow | — | If | # Article to Multi-Platform Social Media Post Generator (full overview note) |
| If | IF | Validate submitted URL | On Article Submission | Scrape Article | # Article to Multi-Platform Social Media Post Generator (full overview note) |
| Scrape Article | HTTP Request | Fetch article HTML | If | Extract Details; ⚠️Unable to Process Article | # Article to Multi-Platform Social Media Post Generator (full overview note) ; ## 📌 Extract Article Content - Title/Content/Author/Excerpt/Site/Image |
| Extract Details | Code | Parse HTML with Readability; extract text/meta/image | Scrape Article | LinkedIn Post Strategist; Twitter Agent; Reddit Form | # Article to Multi-Platform Social Media Post Generator (full overview note) ; ## 📌 Extract Article Content - Title/Content/Author/Excerpt/Site/Image |
| ⚠️Unable to Process Article | Form (completion) | User-friendly failure message | Scrape Article (error output) | — | # Article to Multi-Platform Social Media Post Generator (full overview note) |
| Sticky Note | Sticky Note | Comment | — | — | ## 📌 Extract Article Content - Title/Content/Author/Excerpt/Site/Image |
| Embedding | OpenAI Embeddings (LangChain) | Provide embeddings for vector retrieval | — | LinkedIn Post Vector Store (embedding input) | ## 📌 LinkedIn |
| LinkedIn Post Vector Store | Supabase Vector Store (LangChain) | Retrieve similar viral posts as a tool | Embedding | LinkedIn Post Strategist (tool) | ## 📌 LinkedIn |
| 5 mini | OpenAI Chat Model (LangChain) | LLM for LinkedIn structured parsing | — | Structured Output Parser (as LLM) | ## 📌 LinkedIn |
| Structured Output Parser | Structured Output Parser (LangChain) | Enforce JSON schema for agent output | 5 mini (LLM) | LinkedIn Post Strategist (parser) | ## 📌 LinkedIn |
| 4.1 | OpenAI Chat Model (LangChain) | LLM for LinkedIn agents | — | LinkedIn agents (as LLM) | ## 📌 LinkedIn |
| 5 | OpenAI Chat Model (LangChain) | LLM for LinkedIn agents | — | LinkedIn agents (as LLM) | ## 📌 LinkedIn |
| LinkedIn Post Strategist | LangChain Agent | RAG analysis using viral post DB | Extract Details; LinkedIn Post Vector Store tool; Structured Output Parser | LinkedIn Post Generator | ## 📌 LinkedIn |
| LinkedIn Post Generator | LangChain Agent | Draft LinkedIn post including URL | LinkedIn Post Strategist | LinkedIn Post Formatter | ## 📌 LinkedIn |
| LinkedIn Post Formatter | LangChain Agent | Clean final post text only | LinkedIn Post Generator | Post Review Form | ## 📌 LinkedIn |
| Post Review Form | Form | User review + feedback | LinkedIn Post Formatter | Post Review Condition | ## 📌 LinkedIn |
| Post Review Condition | IF | Decide regenerate vs continue | Post Review Form | Re-generate LinkedIn Post; Download Image for Form | ## 📌 LinkedIn |
| Re-generate LinkedIn Post | LangChain Agent | Rewrite post from feedback | Post Review Condition | Post 2 Review Form | ## 📌 LinkedIn |
| Post 2 Review Form | Form | Second review step | Re-generate LinkedIn Post | Post 2 Review Conditions | ## 📌 LinkedIn |
| Post 2 Review Conditions | IF | Stop after 1 regeneration | Post 2 Review Form | Download Image for Form; The End | ## 📌 LinkedIn |
| The End | Form (completion) | Informs regeneration limit reached | Post 2 Review Conditions | — | ## 📌 LinkedIn |
| Download Image for Form | HTTP Request | Download extracted image (preview step) | Post Review Condition / Post 2 Review Conditions | Image Preview Form | ## 📌 LinkedIn |
| Image Preview Form | Form | Ask whether to use image | Download Image for Form | Image Preview Conditions | ## 📌 LinkedIn |
| Image Preview Conditions | IF | Choose image vs link post | Image Preview Form | Download Image for LinkedIn Post; Text + Link | ## 📌 LinkedIn |
| Download Image for LinkedIn Post | HTTP Request | Download image binary for LinkedIn upload | Image Preview Conditions | Text + Image | ## 📌 LinkedIn |
| Text + Image | LinkedIn | Publish image post | Download Image for LinkedIn Post | — | ## 📌 LinkedIn |
| Text + Link | LinkedIn | Publish link preview post | Image Preview Conditions | — | ## 📌 LinkedIn |
| Structured Output Parser X | Structured Output Parser (LangChain) | Enforce tweet JSON schema | 5 minii (LLM) | Twitter Agent (parser) | ## 📌 X (Twitter) |
| 5 minii | OpenAI Chat Model (LangChain) | LLM for Twitter structured parsing | — | Structured Output Parser X | ## 📌 X (Twitter) |
| 4.i | OpenAI Chat Model (LangChain) | LLM for Twitter agents | — | Twitter Agent; Re-generate Tweet Agent | ## 📌 X (Twitter) |
| 5. | OpenAI Chat Model (LangChain) | LLM for Twitter agents | — | Twitter Agent; Re-generate Tweet Agent | ## 📌 X (Twitter) |
| Twitter Agent | LangChain Agent | Generate tweet under 280 with URL | Extract Details; Structured Output Parser X | Tweet Review Form | ## 📌 X (Twitter) |
| Tweet Review Form | Form | User review + feedback | Twitter Agent | Tweeet Review Conditions | ## 📌 X (Twitter) |
| Tweeet Review Conditions | IF | Decide post vs regenerate | Tweet Review Form | Tweet; Re-generate Tweet Agent | ## 📌 X (Twitter) |
| Re-generate Tweet Agent | LangChain Agent | Rewrite tweet from feedback | Tweeet Review Conditions | Tweet 2 Review Form | ## 📌 X (Twitter) |
| Tweet 2 Review Form | Form | Second review step | Re-generate Tweet Agent | Tweet 2 Review Conditions | ## 📌 X (Twitter) |
| Tweet 2 Review Conditions | IF | Intended gating for second regen | Tweet 2 Review Form | (unused); Re-generated Tweet | ## 📌 X (Twitter) |
| Tweet | Twitter | Publish tweet | Tweeet Review Conditions | — | ## 📌 X (Twitter) |
| Re-generated Tweet | Twitter | Publish regenerated tweet | Tweet 2 Review Conditions | — | ## 📌 X (Twitter) |
| Reddit Form | Form | Choose subreddit | Extract Details | Get Flair | ## 📌 Reddit |
| Get Flair | HTTP Request | Fetch subreddit flairs | Reddit Form | [Flair] | ## 📌 Reddit |
| [Flair] | Aggregate | Aggregate flair texts | Get Flair | Flair Selector Agent | ## 📌 Reddit |
| Flair Selector Agent | OpenAI (prompt) | Pick best flair name from list | [Flair] | Reddit Post | ## 📌 Reddit |
| Reddit Post | HTTP Request | Submit link post to Reddit | Flair Selector Agent | — | ## 📌 Reddit |
| Sticky Note10 | Sticky Note | Comment | — | — | ## 📌 LinkedIn |
| Sticky Note4 | Sticky Note | Comment | — | — | ## 📌 X (Twitter) |
| Sticky Note5 | Sticky Note | Comment | — | — | ## 📌 Reddit |
| Sticky Note1 | Sticky Note | Comment / project overview | — | — | (content itself is the note) |

---

## 4. Reproducing the Workflow from Scratch

1) **Create Form Trigger**
   1. Add node: **Form Trigger**
   2. Path: `article-post`
   3. Add field:
      - Type: Textarea
      - Label: `Article Link`
      - Required: true
   4. Enable **Basic Auth** and select/create credentials (username/password).
   5. Response mode: **Last node**.

2) **Add URL validation**
   1. Add node: **IF**
   2. Condition (Boolean equals):
      - Left: `{{ $json['Article Link'].isUrl() }}`
      - Right: `true`
   3. Connect: Form Trigger → IF.
   4. (Recommended) Add a **Form completion** node for invalid URL and connect IF false to it.

3) **Scrape the article**
   1. Add node: **HTTP Request**
   2. URL: `{{ $json['Article Link'] }}`
   3. Enable **Retry on fail**
   4. Set **On Error** to “Continue (error output)”
   5. Connect IF true → HTTP Request.
   6. Add **Form (completion)** node titled “⚠️Unable to Process Article” with message “Please enter different Article”, connect HTTP Request error output to it.

4) **Extract article details (Readability)**
   1. Add node: **Code**
   2. Paste logic that:
      - Reads HTML from input (`data`) and URL (`url`)
      - Uses `jsdom` + `@mozilla/readability`
      - Extracts: `title`, `byline/author`, `textContent`, `excerpt`, `siteName`, image selection
   3. Connect HTTP Request success output → Code.
   4. Ensure your n8n runtime has `jsdom` and `@mozilla/readability` available (self-host: install in the environment/container).

5) **LinkedIn RAG components**
   1. Add node: **OpenAI Embeddings (LangChain)** and configure OpenAI credentials.
   2. Add node: **Supabase Vector Store (LangChain)**
      - Mode: “Retrieve as tool”
      - Table: `linkedin_post`
      - Configure Supabase credentials (URL + API key).
   3. Connect Embeddings → Vector Store (embedding input).

6) **LinkedIn strategist agent**
   1. Add node: **Structured Output Parser (LangChain)** with schema example:
      - `{ "Post Content": "Post Content" }`
      - Enable Auto-fix
   2. Add one or more **OpenAI Chat Model (LangChain)** nodes as LLM providers (e.g., `gpt-4.1`, `gpt-5`, `gpt-5-mini`) and connect them to:
      - Parser (LLM input)
      - Agents (LLM input)
   3. Add node: **AI Agent (LangChain)** named “LinkedIn Post Strategist”
      - Prompt: include the extracted fields (`{{$json.title}}`, etc.)
      - System message: your strategist instructions
      - Attach Vector Store as **tool**
      - Attach Structured Output Parser as **output parser**
   4. Connect: Extract Details → LinkedIn Post Strategist.

7) **LinkedIn generator + formatter**
   1. Add node: **AI Agent** “LinkedIn Post Generator”
      - Include article text + URL in the prompt
      - Include strategist output in the system/prompt (ensure consistent field names)
      - Rule: must include URL
   2. Add node: **AI Agent** “LinkedIn Post Formatter”
      - Input: generator output
      - Rules: post-only, no markdown, no em dashes, apply Sans Serif Bold Unicode emphasis
   3. Connect: Strategist → Generator → Formatter.

8) **LinkedIn review loop**
   1. Add node: **Form** “Post Review Form” that renders `{{$json.output}}` as HTML and collects:
      - Radio: regenerate? (Yes/No)
      - Textarea: feedback
   2. Add node: **IF** “Post Review Condition” checking regenerate == Yes.
   3. Add node: **AI Agent** “Re-generate LinkedIn Post” using original formatted post + feedback.
   4. Add node: **Form** “Post 2 Review Form” with radio “regenerate again?”.
   5. Add node: **IF** “Post 2 Review Conditions” to allow only one regeneration.
   6. Add node: **Form completion** “The End” for the “try again” scenario.
   7. Wire:
      - Formatter → Post Review Form → IF
      - IF true → Re-generate → Post 2 Review Form → IF
      - IF false OR Post2 IF true → proceed to image step
      - Post2 IF false → The End

9) **LinkedIn image preview + publish**
   1. Add node: **HTTP Request** “Download Image for Form” with URL `{{ $('Extract Details').item.json.imageUrl }}`
   2. Add node: **Form** “Image Preview Form” showing a link to the image and radio “Use image?” (Yes / Continue without Image)
   3. Add node: **IF** “Image Preview Conditions” (Yes/No)
   4. Add node: **HTTP Request** “Download Image for LinkedIn Post” with same image URL (binary)
   5. Add node: **LinkedIn** “Text + Image”
      - share type: IMAGE
      - text: `{{ $('LinkedIn Post Formatter').item.json.output ?? $('Re-generate LinkedIn Post').item.json.output }}`
      - configure LinkedIn OAuth2 credentials
      - set `person` to your LinkedIn identifier as required by the node
   6. Add node: **LinkedIn** “Text + Link”
      - share type: ARTICLE
      - text: same expression
      - `originalUrl`: submitted article link
      - `title`: extracted title
   7. Wire:
      - (from review accepted) → Download Image for Form → Image Preview Form → IF
      - IF true → Download Image for LinkedIn Post → Text + Image
      - IF false → Text + Link
   8. (Recommended) Add a guard: if `imageUrl` is null, skip image preview and go directly to Text + Link.

10) **Twitter/X generation + review + tweet**
   1. Add **Structured Output Parser** (schema `{ "Post Content": "Post Content" }`, autoFix on).
   2. Add **LangChain AI Agent** “Twitter Agent”
      - Prompt includes extracted fields + article URL
      - Rules: < 280 chars, must include URL
      - Attach the Twitter parser
   3. Add **Form** “Tweet Review Form” displaying `{{$json.output['Post Content']}}`
   4. Add **IF** “Tweet Review Conditions” to branch to posting vs regeneration
   5. Add **AI Agent** “Re-generate Tweet Agent” (uses original tweet + feedback)
   6. Add **Form** “Tweet 2 Review Form” + **IF** “Tweet 2 Review Conditions”
   7. Add **Twitter** node(s) to post:
      - Text from agent output
      - Configure Twitter OAuth2 credentials
   8. Important fixes you should apply when recreating:
      - Ensure the IF checks the **exact** form field name (avoid the “again??” mismatch).
      - If you want media attachments, add a **Twitter media upload** step; otherwise remove `attachments` mapping.

11) **Reddit posting**
   1. Add **Form** “Reddit Form” with dropdown `Subreddit` options.
   2. Add **HTTP Request** “Get Flair”
      - URL: `https://oauth.reddit.com/r/{{ $json.Subreddit.split("/")[1] }}/api/link_flair_v2`
      - OAuth2 credentials for Reddit
   3. Add **Aggregate** node to combine flair texts into one list/string.
   4. Add **OpenAI** node to select one flair (plain text).
   5. Add **HTTP Request** “Reddit Post”
      - Prefer URL: `https://oauth.reddit.com/api/submit` (not hardcoded subreddit path)
      - Send required params:
        - `sr`: `{{ $json.Subreddit.split("/")[1] }}`
        - `kind`: `link`
        - `title`: extracted title
        - `url`: article URL
      - If you want flair:
        - Convert selected flair text to flair_id (requires mapping from Get Flair response), then pass `flair_id`
   6. (Recommended) Remove hardcoded `flair_id` and hardcoded `/r/n8n/` path; actually use selected subreddit and selected flair.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “Article to Multi-Platform Social Media Post Generator” overview and operating notes (extraction, LinkedIn 3-agent system with RAG, Twitter rules, Reddit flair selection, credentials list, setup steps) | Included as a large sticky note in the workflow canvas (project-wide context). |
| Disclaimer: “Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n…” | Provided by the user alongside the workflow; applies to the whole document/context. |