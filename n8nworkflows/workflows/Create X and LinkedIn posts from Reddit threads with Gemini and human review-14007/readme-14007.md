Create X and LinkedIn posts from Reddit threads with Gemini and human review

https://n8nworkflows.xyz/workflows/create-x-and-linkedin-posts-from-reddit-threads-with-gemini-and-human-review-14007


# Create X and LinkedIn posts from Reddit threads with Gemini and human review

# 1. Workflow Overview

This workflow takes a Reddit thread URL submitted through an n8n form, retrieves the thread via Reddit’s public JSON API, extracts the post and comment discussion, summarizes it with Google Gemini, generates platform-specific content for X and LinkedIn, then pauses for human review before optionally publishing to both networks.

Its primary use cases are:
- Turning Reddit discussions into social-ready content
- Accelerating social content research and drafting
- Keeping a human approval step before publishing
- Supporting optional manual edits without changing the AI generation flow

## 1.1 Input Reception and URL Normalization
The workflow starts with a form where a user submits a Reddit thread URL. A code node validates the URL and derives Reddit API-friendly values such as subreddit, post ID, and `.json` endpoint.

## 1.2 Reddit Data Retrieval and Structuring
The workflow calls Reddit’s public JSON API, then parses the response into:
- post metadata
- a nested comment tree
- a flattened AI-friendly text representation of the thread

## 1.3 AI Summarization
Gemini receives the structured thread text and produces a markdown summary covering overview, key topics, notable insights, sentiment, and actionable takeaways.

## 1.4 AI Social Post Generation
A second Gemini node converts the summary into two drafts:
- one X post
- one LinkedIn post

The expected return format is strict JSON.

## 1.5 Response Parsing and Validation
A code node extracts the Gemini output, removes code fences if needed, parses the JSON, validates required keys, and enforces the X character limit.

## 1.6 Human Review and Final Content Resolution
A Wait/Form node pauses the workflow and displays both generated posts. The reviewer can approve, reject, or override either post. A code node merges overrides with AI output.

## 1.7 Decision and Publishing
An IF node checks the reviewer’s decision:
- if approved, the workflow posts to X and LinkedIn
- if rejected, it ends gracefully with a rejection status message

---

# 2. Block-by-Block Analysis

## Block 1: Form Input and Reddit URL Parsing

### Overview
This block collects a Reddit thread URL from the user and transforms it into a normalized, API-ready structure. It is the workflow’s only entry point and ensures downstream nodes receive a valid Reddit thread reference.

### Nodes Involved
- Reddit URL Input
- Parse Reddit URL

### Node Details

#### 1. Reddit URL Input
- **Type and technical role:** `n8n-nodes-base.formTrigger`  
  Entry-point trigger node that exposes a hosted form and starts the workflow on submission.
- **Configuration choices:**
  - Form title: `Reddit Thread Post Generator`
  - Description: indicates the form generates social media posts from a Reddit thread
  - One field: `Reddit Thread URL`
- **Key expressions or variables used:** None in parameters
- **Input and output connections:**
  - Input: none
  - Output: `Parse Reddit URL`
- **Version-specific requirements:** Uses `typeVersion: 2.5`; requires form trigger support in the installed n8n version
- **Edge cases or potential failure types:**
  - Empty field submission
  - Invalid Reddit URL format passed downstream
  - Form accessibility/webhook URL issues if n8n instance is not publicly reachable
- **Sub-workflow reference:** None

#### 2. Parse Reddit URL
- **Type and technical role:** `n8n-nodes-base.code`  
  JavaScript transformation node that validates and parses the submitted Reddit URL.
- **Configuration choices:**
  - Reads `Reddit Thread URL` from the form output
  - Trims whitespace
  - Normalizes scheme and removes `www.` / `m.` prefixes
  - Uses regex to extract:
    - subreddit
    - post ID
  - Builds:
    - `apiUrl`
    - `cleanThreadUrl`
    - original URL copy
- **Key expressions or variables used:**
  - `$input.first().json['Reddit Thread URL']`
  - Output fields:
    - `originalUrl`
    - `subreddit`
    - `postId`
    - `apiUrl`
    - `cleanThreadUrl`
- **Input and output connections:**
  - Input: `Reddit URL Input`
  - Output: `Fetch Reddit Thread`
- **Version-specific requirements:** `typeVersion: 2`
- **Edge cases or potential failure types:**
  - Throws explicit error for invalid standard Reddit thread URLs
  - The comment says it supports `redd.it`, but the actual regex only matches full `reddit.com/r/.../comments/...` URLs; short links are not actually handled by this implementation
  - Missing or malformed form field name would break expression access
- **Sub-workflow reference:** None

---

## Block 2: Reddit Thread Retrieval and Content Extraction

### Overview
This block fetches the Reddit thread data and transforms raw Reddit JSON into a structured representation usable by both humans and AI. It extracts metadata and recursively traverses comments to create a flat prompt string.

### Nodes Involved
- Fetch Reddit Thread
- Extract Thread Content

### Node Details

#### 3. Fetch Reddit Thread
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Performs the HTTP GET request against Reddit’s public JSON thread endpoint.
- **Configuration choices:**
  - URL: `={{ $json.apiUrl }}`
  - Query parameters:
    - `limit=100`
    - `depth=3`
  - Header:
    - `User-Agent: n8n-workflow-automation/1.0`
  - Sends query and headers explicitly
- **Key expressions or variables used:**
  - `{{ $json.apiUrl }}`
- **Input and output connections:**
  - Input: `Parse Reddit URL`
  - Output: `Extract Thread Content`
- **Version-specific requirements:** `typeVersion: 4.2`
- **Edge cases or potential failure types:**
  - Reddit rate limiting or temporary blocking
  - Invalid or deleted thread causing non-useful response content
  - NSFW/quarantined/private/community-restricted threads may not be accessible
  - Network timeout or connectivity issues
  - Reddit may return truncated discussion because `more` nodes are not expanded later
- **Sub-workflow reference:** None

#### 4. Extract Thread Content
- **Type and technical role:** `n8n-nodes-base.code`  
  Parses Reddit API response into post metadata, nested comments, statistics, and a flattened AI-ready text block.
- **Configuration choices:**
  - Handles either:
    - API response as one array item
    - multiple items from the HTTP node
  - Extracts post metadata such as:
    - title
    - body
    - author
    - score
    - upvote ratio
    - subreddit
    - comment count
    - permalink
    - link/self-post information
    - flair
    - awards
  - Recursively traverses comment replies
  - Sorts siblings by score descending at every level
  - Skips:
    - `more` placeholders
    - deleted/removed/empty comments
  - Builds:
    - `structuredComments`
    - `totalExtracted`
    - `maxDepthFound`
    - `threadContent`
- **Key expressions or variables used:**
  - `$input.first().json`
  - `$input.all()`
  - Output fields include:
    - `structuredComments`
    - `threadContent`
    - post metadata fields like `title`, `body`, `threadUrl`, `numComments`
- **Input and output connections:**
  - Input: `Fetch Reddit Thread`
  - Output: `Summarize Thread`
- **Version-specific requirements:** `typeVersion: 2`
- **Edge cases or potential failure types:**
  - Throws if Reddit response shape is unexpected
  - Can miss comments hidden behind Reddit `more` stubs because no secondary expansion call is made
  - Deep or large threads can create very large prompt payloads
  - Deleted post bodies or removed comments reduce available context
  - `maxDepthFound` is inferred from indentation in generated lines; functional but indirect
- **Sub-workflow reference:** None

---

## Block 3: AI Summarization

### Overview
This block converts the extracted Reddit discussion into a structured markdown summary. It provides the strategic intermediate representation used by the social copy generator.

### Nodes Involved
- Summarize Thread

### Node Details

#### 5. Summarize Thread
- **Type and technical role:** `@n8n/n8n-nodes-langchain.googleGemini`  
  Invokes Google Gemini to summarize the Reddit thread.
- **Configuration choices:**
  - Model: `models/gemini-3.1-flash-lite-preview`
  - Prompt asks for markdown summary with sections:
    - Thread Overview
    - Key Topics Discussed
    - Notable Insights & Perspectives
    - Community Sentiment
    - Actionable Takeaways
  - Injects `threadContent`
  - Options:
    - `topK: 20`
    - `temperature: 0.7`
    - `maxToolsIterations: 5`
  - Built-in tool enabled:
    - `urlContext: true`
  - `simplify: false`, so raw Gemini-style response structure is preserved
- **Key expressions or variables used:**
  - `{{ $json.threadContent }}`
- **Input and output connections:**
  - Input: `Extract Thread Content`
  - Output: `Generate Social Posts`
- **Version-specific requirements:**
  - Requires the LangChain Gemini node package installed
  - Requires valid `Google Gemini(PaLM) Api account` credentials
  - Uses `typeVersion: 1.1`
- **Edge cases or potential failure types:**
  - Authentication or quota failures on Google Gemini
  - Long Reddit threads may exceed model input limits
  - Model may return incomplete or oddly structured output if prompt/token budget is constrained
  - Preview model IDs may change or be deprecated over time
- **Sub-workflow reference:** None

---

## Block 4: AI Social Post Generation

### Overview
This block turns the markdown summary into platform-specific social content. It instructs Gemini to output strict raw JSON with one X post and one LinkedIn post.

### Nodes Involved
- Generate Social Posts

### Node Details

#### 6. Generate Social Posts
- **Type and technical role:** `@n8n/n8n-nodes-langchain.googleGemini`  
  Uses Gemini to generate social posts from the prior summary.
- **Configuration choices:**
  - Model: `models/gemini-3.1-flash-lite-preview`
  - Prompt positions the model as an expert social media strategist
  - Uses summary text from the previous Gemini node
  - Requires exact JSON shape:
    - `x_post`
    - `linkedin_post`
  - No markdown fences or explanatory text allowed
- **Key expressions or variables used:**
  - `{{ $json.candidates[0].content.parts[0].text }}`
- **Input and output connections:**
  - Input: `Summarize Thread`
  - Output: `Parse Social Posts`
- **Version-specific requirements:**
  - Same Gemini credential requirement as above
  - `typeVersion: 1.1`
- **Edge cases or potential failure types:**
  - Depends on raw response structure from previous node because `simplify` is disabled there
  - If the prior node output shape changes, this expression may fail
  - Model may still return fenced JSON or malformed JSON despite instructions
  - X post may exceed 280 characters; downstream parser truncates it
- **Sub-workflow reference:** None

---

## Block 5: Social JSON Parsing and Validation

### Overview
This block normalizes Gemini’s output into clean workflow fields. It is designed defensively to handle multiple possible Gemini response shapes and malformed JSON.

### Nodes Involved
- Parse Social Posts

### Node Details

#### 7. Parse Social Posts
- **Type and technical role:** `n8n-nodes-base.code`  
  Extracts text from Gemini output, parses JSON, validates required fields, and enforces X length.
- **Configuration choices:**
  - Supports multiple response formats:
    - array of candidates
    - single candidate object
    - `{ text: '...' }`
    - nested `candidates`
  - Removes code fences like ```json
  - Tries `JSON.parse`
  - Falls back to regex extraction for `x_post` and `linkedin_post`
  - Trims content
  - Truncates X text to 280 chars using `277 + '...'`
- **Key expressions or variables used:**
  - `$input.first().json`
  - Output fields:
    - `x_post`
    - `linkedin_post`
    - `x_char_count`
- **Input and output connections:**
  - Input: `Generate Social Posts`
  - Output: `Human Approval`
- **Version-specific requirements:** `typeVersion: 2`
- **Edge cases or potential failure types:**
  - Throws if no recognizable text field is present
  - Throws if JSON cannot be parsed and regex fallback also fails
  - Throws if parsed JSON lacks `x_post` or `linkedin_post`
  - Character counting is simple string length; not tailored for every Unicode/display nuance on X
- **Sub-workflow reference:** None

---

## Block 6: Human Review and Override Resolution

### Overview
This block introduces a human-in-the-loop approval checkpoint. The workflow pauses, displays both generated posts, captures approval or rejection, and merges any manual edits with the AI output.

### Nodes Involved
- Human Approval
- Resolve Final Posts

### Node Details

#### 8. Human Approval
- **Type and technical role:** `n8n-nodes-base.wait`  
  Pauses execution and resumes via a form submission.
- **Configuration choices:**
  - Resume mode: `form`
  - Form title: `📋 Review & Approve Social Media Posts`
  - Description dynamically displays:
    - current X draft
    - X character count
    - current LinkedIn draft
  - Form fields:
    - `X Post Override (leave blank to use AI version)` textarea
    - `LinkedIn Post Override (leave blank to use AI version)` textarea
    - `Decision` dropdown with:
      - `Approve & Publish`
      - `Reject`
  - `Decision` is required
- **Key expressions or variables used:**
  - `{{ $json.x_char_count }}`
  - `{{ $json.x_post }}`
  - `{{ $json.linkedin_post }}`
- **Input and output connections:**
  - Input: `Parse Social Posts`
  - Output: `Resolve Final Posts`
- **Version-specific requirements:** `typeVersion: 1.1`
- **Edge cases or potential failure types:**
  - Resume webhook/form must remain reachable
  - Long waiting periods may be affected by workflow retention/execution settings
  - User may submit blank overrides, which is expected
  - If someone manually enters empty-looking but whitespace-only overrides, code trims them out
- **Sub-workflow reference:** None

#### 9. Resolve Final Posts
- **Type and technical role:** `n8n-nodes-base.code`  
  Merges human overrides with the AI-generated drafts after the Wait node resumes.
- **Configuration choices:**
  - Reads resumed form submission from `$input.first().json`
  - Explicitly references upstream node output:
    - `$('Parse Social Posts').item.json`
  - Logic:
    - override wins if non-empty
    - otherwise AI version is used
  - Returns:
    - `X Post`
    - `LinkedIn Post`
    - `usedOverrideX`
    - `usedOverrideLi`
- **Key expressions or variables used:**
  - `$input.first().json`
  - `$('Parse Social Posts').item.json`
  - Form fields:
    - `X Post Override (leave blank to use AI version)`
    - `LinkedIn Post Override (leave blank to use AI version)`
- **Input and output connections:**
  - Input: `Human Approval`
  - Output: `Approved?`
- **Version-specific requirements:** `typeVersion: 2`
- **Edge cases or potential failure types:**
  - If node name `Parse Social Posts` changes, the explicit reference will break
  - Throws if final X or LinkedIn post resolves to empty
  - If execution data from prior node is unavailable due to retention or unusual execution mode, lookup could fail
- **Sub-workflow reference:** None

---

## Block 7: Approval Routing and Publishing

### Overview
This final block evaluates the reviewer’s decision and either publishes both posts or exits cleanly without publishing. It is the enforcement point for human-controlled release.

### Nodes Involved
- Approved?
- Post to X
- Post to Linkedin
- Rejected – End

### Node Details

#### 10. Approved?
- **Type and technical role:** `n8n-nodes-base.if`  
  Branches execution based on approval form data.
- **Configuration choices:**
  - Compares `Decision` from `Human Approval`
  - Condition: equals `Approve & Publish`
  - True branch publishes
  - False branch rejects
- **Key expressions or variables used:**
  - `={{ $('Human Approval').item.json['Decision'] }}`
- **Input and output connections:**
  - Input: `Resolve Final Posts`
  - True output: `Post to X`, `Post to Linkedin`
  - False output: `Rejected – End`
- **Version-specific requirements:** `typeVersion: 2`
- **Edge cases or potential failure types:**
  - Depends on exact node name `Human Approval`
  - Depends on exact dropdown value `Approve & Publish`
  - Renaming form field or option text breaks condition logic
- **Sub-workflow reference:** None

#### 11. Post to X
- **Type and technical role:** `n8n-nodes-base.twitter`  
  Publishes the final X post using X/Twitter OAuth2 credentials.
- **Configuration choices:**
  - Text: `={{ $json['X Post'] }}`
- **Key expressions or variables used:**
  - `{{ $json['X Post'] }}`
- **Input and output connections:**
  - Input: `Approved?` true branch
  - Output: none
- **Version-specific requirements:**
  - Requires configured `X account` OAuth2 credentials
  - Uses `typeVersion: 2`
- **Edge cases or potential failure types:**
  - Auth/token issues
  - App permission limitations
  - Duplicate content restrictions or policy-related API rejection
  - Effective platform character rules may still reject certain content despite length truncation
- **Sub-workflow reference:** None

#### 12. Post to Linkedin
- **Type and technical role:** `n8n-nodes-base.linkedIn`  
  Publishes the final LinkedIn post.
- **Configuration choices:**
  - Text: `={{ $json['LinkedIn Post'] }}`
  - Person: `ID`
- **Key expressions or variables used:**
  - `{{ $json['LinkedIn Post'] }}`
- **Input and output connections:**
  - Input: `Approved?` true branch
  - Output: none
- **Version-specific requirements:**
  - Requires configured `LinkedIn account` OAuth2 credentials
  - Uses `typeVersion: 1`
  - The `person` field must be set correctly for the authenticated profile/account context
- **Edge cases or potential failure types:**
  - OAuth permission issues
  - Invalid author/person ID configuration
  - LinkedIn API restrictions on posting scope
  - Post formatting may differ from preview due to LinkedIn rendering
- **Sub-workflow reference:** None

#### 13. Rejected – End
- **Type and technical role:** `n8n-nodes-base.set`  
  Creates a terminal structured output for the rejection path.
- **Configuration choices:**
  - Sets:
    - `status = rejected`
    - `message = Posts were reviewed and rejected by the user. No content was published.`
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Input: `Approved?` false branch
  - Output: none
- **Version-specific requirements:** `typeVersion: 3.4`
- **Edge cases or potential failure types:**
  - Minimal risk; primarily a terminal bookkeeping node
- **Sub-workflow reference:** None

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Reddit URL Input | Form Trigger | Entry form for Reddit thread URL submission |  | Parse Reddit URL | ## 📥 Reddit URL Input<br>**Type:** Form Trigger<br>**Purpose:** Entry point — presents a web form to collect a Reddit thread URL from the user.<br>**Input:** User submits a Reddit thread URL via the form.<br>**Output:** Raw URL string forwarded to the parse step. |
| Parse Reddit URL | Code | Validate and decompose Reddit URL into API-ready fields | Reddit URL Input | Fetch Reddit Thread | ## 🔍 Parse Reddit URL<br>**Type:** Code (JavaScript)<br>**Purpose:** Validates and deconstructs the submitted Reddit URL.<br>**Input:** Raw URL string<br>**Function:** Regex extraction of subreddit + post ID. Handles standard, mobile (`m.`) and short `redd.it` links.<br>**Output:** `subreddit`, `postId`, `apiUrl`, `cleanThreadUrl` |
| Fetch Reddit Thread | HTTP Request | Retrieve Reddit thread JSON from public API | Parse Reddit URL | Extract Thread Content | ## 🌐 Fetch Reddit Thread<br>**Type:** HTTP Request<br>**Purpose:** Retrieves the full thread via Reddit's public JSON API.<br>**Input:** `apiUrl` (e.g. `reddit.com/r/sub/comments/id.json`)<br>**Function:** GET with `limit=100`, `depth=3`.<br>**Output:** Raw Reddit API response — post listing + comments listing array. |
| Extract Thread Content | Code | Extract metadata, comments, and AI-ready thread text | Fetch Reddit Thread | Summarize Thread | ## ⚙️ Extract Content<br>**Type:** Code (JavaScript)<br>**Purpose:** Parses raw Reddit JSON into structured, AI-ready content.<br>**Input:** Raw Reddit API response<br>**Function:** Extracts post metadata; recursively traverses all comments sorted by score at every depth; builds a flat indented string for AI.<br>**Output:** `threadContent` (AI prompt string), `structuredComments` (nested JSON), post metadata |
| Summarize Thread | Google Gemini | Produce markdown summary of thread discussion | Extract Thread Content | Generate Social Posts | ## 🧠 Summarize Thread<br>**Type:** Google Gemini AI<br>**Purpose:** Produces a comprehensive markdown summary of the Reddit thread.<br>**Input:** `threadContent` from Extract node<br>**Function:** Gemini 3.1 Flash Lite → returns Thread Overview, Key Topics, Notable Insights, Community Sentiment, and Actionable Takeaways.<br>**Output:** Structured markdown summary text |
| Generate Social Posts | Google Gemini | Generate X and LinkedIn post drafts from summary | Summarize Thread | Parse Social Posts | ## ✍️ Generate Social Posts<br>**Type:** Google Gemini AI<br>**Purpose:** Creates platform-optimized posts for X and LinkedIn.<br>**Input:** Thread summary from Summarize node<br>**Function:** Gemini generates: X post (≤280 chars, punchy + hashtags) and LinkedIn post (150–300 words, professional + hashtags). Returns strict JSON only.<br>**Output:** `{ "x_post": "...", "linkedin_post": "..." }` |
| Parse Social Posts | Code | Parse and validate AI JSON response for social posts | Generate Social Posts | Human Approval | ## 🧹 Parse Social Posts<br>**Type:** Code (JavaScript)<br>**Purpose:** Safely extracts and validates Gemini's JSON response.<br>**Input:** Raw Gemini API response<br>**Function:** Strips markdown fences, parses JSON, falls back to regex if malformed. Enforces 280-char hard limit on X post.<br>**Output:** `x_post`, `linkedin_post`, `x_char_count` |
| Human Approval | Wait | Pause for manual review, approval, or override | Parse Social Posts | Resolve Final Posts | ## 👤 Human Approval<br>**Type:** Wait (Form)<br>**Purpose:** Pauses the workflow so a human can review posts.<br>**Input:** `x_post`, `linkedin_post`, `x_char_count`<br>**Function:** Displays posts in a form. User can overrides or use AI version. Must select **Approve & Publish** or **Reject**.<br>**Output:** Form submission with `Decision` + optional override fields. |
| Resolve Final Posts | Code | Merge manual overrides with AI-generated posts | Human Approval | Approved? | ## ✅ Resolve Final Posts<br>**Type:** Code (JavaScript)<br>**Purpose:** Merges human overrides with AI-generated posts.<br>**Input:** Form data + AI posts (referenced from Parse Social Posts node)<br>**Function:** Override wins if non-empty; otherwise falls back to the AI version. Throws if both are empty.<br>**Output:** `X Post`, `LinkedIn Post` (final content ready to publish) |
| Approved? | IF | Route based on approval decision | Resolve Final Posts | Post to X; Post to Linkedin; Rejected – End | ## 🔀 Approved?<br>**Type:** IF Condition<br>**Purpose:** Routes the workflow based on the human's decision.<br>**Input:** `Decision` field from Human Approval form<br>**True →** Post to X + Post to LinkedIn<br>**False →** Rejected – End (nothing published) |
| Post to X | X (Twitter) | Publish final post to X | Approved? |  | ## 🐦 Post to X<br>**Type:** X (Twitter)<br>**Purpose:** Publishes the final post to X (Twitter).<br>**Input:** `X Post` (≤280 chars) from Resolve Final Posts<br>**Output:** Live tweet on the connected X account. |
| Post to Linkedin | LinkedIn | Publish final post to LinkedIn | Approved? |  | ## 💼 Post to LinkedIn<br>**Type:** LinkedIn<br>**Purpose:** Publishes the final post to LinkedIn.<br>**Input:** `LinkedIn Post` from Resolve Final Posts<br>**Output:** Live post on the connected LinkedIn account. |
| Rejected – End | Set | Terminal rejection output without publishing | Approved? |  | ## 🚫 Rejected – End<br>**Type:** Set<br>**Purpose:** Graceful termination — nothing is published.<br>**Input:** Rejection path from Approved? node<br>**Function:** Sets `status: 'rejected'` and a descriptive message.<br>**Output:** Terminal state — workflow ends here. |
| 🗒️ Workflow Summary | Sticky Note | Documentation note |  |  | ## 🔁 Reddit → Social Media Automation<br>Converts any Reddit thread into ready-to-publish posts for **X (Twitter)** and **LinkedIn** — with a human review gate before anything goes live.<br>**📌 Data Flow:**<br>Reddit URL → Parse → Fetch API → Extract Content → AI Summarize → Generate Posts → Human Approval → Publish<br>**✨ Key Features:**<br>- Accepts any Reddit thread URL<br>- Extracts post + all nested comments (sorted by score)<br>- AI-powered thread summarization via Gemini<br>- Platform-optimized X + LinkedIn post generation<br>- Human-in-the-loop approval with override support<br>- Graceful rejection path — nothing published without approval<br>**🛠️ Tools Used:**<br>Google Gemini 3.1 Flash Lite · Reddit JSON API · X (Twitter) API · LinkedIn API |
| 🌐 Node: Fetch Reddit Thread | Sticky Note | Documentation note |  |  |  |
| Sticky Note1 | Sticky Note | Documentation note |  |  |  |
| Sticky Note2 | Sticky Note | Documentation note |  |  |  |
| Sticky Note3 | Sticky Note | Documentation note |  |  |  |
| Sticky Note4 | Sticky Note | Documentation note |  |  |  |
| Sticky Note5 | Sticky Note | Documentation note |  |  |  |
| Sticky Note6 | Sticky Note | Documentation note |  |  |  |
| Sticky Note7 | Sticky Note | Documentation note |  |  |  |
| Sticky Note8 | Sticky Note | Documentation note |  |  |  |
| Sticky Note9 | Sticky Note | Documentation note |  |  |  |
| Sticky Note10 | Sticky Note | Documentation note |  |  |  |
| Sticky Note11 | Sticky Note | Documentation note |  |  |  |
| Sticky Note | Sticky Note | Documentation note |  |  |  |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it something like: `Reddit Thread → AI Summary → Social Media Publisher (X + LinkedIn)`.
   - Keep execution order at the default compatible mode used by your n8n version.

2. **Add the Form Trigger node**
   - Node type: `Form Trigger`
   - Name: `Reddit URL Input`
   - Form title: `Reddit Thread Post Generator`
   - Form description: `generates social media posts based on reddit thread`
   - Add one form field:
     - Label: `Reddit Thread URL`
   - This is the workflow entry point.

3. **Add a Code node after the form**
   - Node type: `Code`
   - Name: `Parse Reddit URL`
   - Paste JavaScript that:
     - reads `Reddit Thread URL`
     - trims it
     - normalizes scheme/domain prefix
     - extracts subreddit and post ID from standard Reddit thread URLs
     - outputs `originalUrl`, `subreddit`, `postId`, `apiUrl`, `cleanThreadUrl`
   - Connect: `Reddit URL Input` → `Parse Reddit URL`

4. **Add an HTTP Request node**
   - Node type: `HTTP Request`
   - Name: `Fetch Reddit Thread`
   - Method: `GET`
   - URL: `={{ $json.apiUrl }}`
   - Enable query parameters:
     - `limit` = `100`
     - `depth` = `3`
   - Enable headers:
     - `User-Agent` = `n8n-workflow-automation/1.0`
   - Connect: `Parse Reddit URL` → `Fetch Reddit Thread`

5. **Add a second Code node for Reddit extraction**
   - Node type: `Code`
   - Name: `Extract Thread Content`
   - Paste JavaScript that:
     - parses Reddit’s listing response
     - extracts thread metadata
     - recursively traverses comments
     - sorts comments by score at each level
     - skips invalid/deleted/removed comments
     - creates:
       - `structuredComments`
       - `threadContent`
       - `totalExtracted`
       - `maxDepthFound`
       - post metadata fields
   - Connect: `Fetch Reddit Thread` → `Extract Thread Content`

6. **Add a Google Gemini node for summarization**
   - Node type: `Google Gemini` from the LangChain-compatible n8n package
   - Name: `Summarize Thread`
   - Credentials: create/select `Google Gemini(PaLM) Api account`
   - Model: `models/gemini-3.1-flash-lite-preview`
   - Set messages with one prompt that:
     - injects `{{ $json.threadContent }}`
     - requests a markdown summary with sections for overview, topics, insights, sentiment, and takeaways
   - Recommended options from this workflow:
     - `topK: 20`
     - `temperature: 0.7`
     - `maxToolsIterations: 5`
   - Set `simplify` to `false`
   - Enable built-in tool `urlContext` if available in your version
   - Connect: `Extract Thread Content` → `Summarize Thread`

7. **Add another Google Gemini node for social generation**
   - Node type: `Google Gemini`
   - Name: `Generate Social Posts`
   - Reuse the same Gemini credentials
   - Model: `models/gemini-3.1-flash-lite-preview`
   - Prompt should:
     - describe the model as an expert social strategist
     - inject the summary from the previous node using:
       `{{ $json.candidates[0].content.parts[0].text }}`
     - ask for:
       - X post under 280 chars with hashtags
       - LinkedIn post 150–300 words with final-line hashtags
     - require raw JSON only in the exact format:
       - `x_post`
       - `linkedin_post`
   - Connect: `Summarize Thread` → `Generate Social Posts`

8. **Add a Code node to parse Gemini JSON**
   - Node type: `Code`
   - Name: `Parse Social Posts`
   - Paste JavaScript that:
     - extracts response text from possible Gemini output shapes
     - strips markdown fences
     - tries `JSON.parse`
     - falls back to regex extraction if needed
     - validates `x_post` and `linkedin_post`
     - truncates X content to 280 chars max
     - outputs:
       - `x_post`
       - `linkedin_post`
       - `x_char_count`
   - Connect: `Generate Social Posts` → `Parse Social Posts`

9. **Add a Wait node configured as a review form**
   - Node type: `Wait`
   - Name: `Human Approval`
   - Resume mode: `Form`
   - Form title: `📋 Review & Approve Social Media Posts`
   - Form description should display the generated drafts using expressions:
     - X post and char count
     - LinkedIn post
   - Add three form fields:
     1. Textarea: `X Post Override (leave blank to use AI version)`
     2. Textarea: `LinkedIn Post Override (leave blank to use AI version)`
     3. Dropdown: `Decision`
        - `Approve & Publish`
        - `Reject`
        - mark as required
   - Connect: `Parse Social Posts` → `Human Approval`

10. **Add a Code node to resolve final content**
    - Node type: `Code`
    - Name: `Resolve Final Posts`
    - Paste JavaScript that:
      - reads form submission from `$input.first().json`
      - references `$('Parse Social Posts').item.json`
      - selects override values if non-empty
      - falls back to AI values otherwise
      - throws if final content is empty
      - outputs:
        - `X Post`
        - `LinkedIn Post`
        - `usedOverrideX`
        - `usedOverrideLi`
    - Important: keep the upstream parser node name exactly `Parse Social Posts`, or update the expression accordingly
    - Connect: `Human Approval` → `Resolve Final Posts`

11. **Add an IF node for approval routing**
    - Node type: `IF`
    - Name: `Approved?`
    - Condition:
      - left value: `={{ $('Human Approval').item.json['Decision'] }}`
      - operator: equals
      - right value: `Approve & Publish`
    - Connect: `Resolve Final Posts` → `Approved?`

12. **Add the X publishing node**
    - Node type: `X` / `Twitter`
    - Name: `Post to X`
    - Credentials: create/select `X account` OAuth2
    - Text field: `={{ $json['X Post'] }}`
    - Connect from the **true** output of `Approved?` to `Post to X`

13. **Add the LinkedIn publishing node**
    - Node type: `LinkedIn`
    - Name: `Post to Linkedin`
    - Credentials: create/select `LinkedIn account` OAuth2
    - Text field: `={{ $json['LinkedIn Post'] }}`
    - Person/account field: configure the correct author context for the authenticated account
    - In the provided workflow this is set to `ID`, which may need replacement with a real profile/person identifier depending on your n8n/LinkedIn setup
    - Connect from the **true** output of `Approved?` to `Post to Linkedin`

14. **Add a Set node for rejection**
    - Node type: `Set`
    - Name: `Rejected – End`
    - Add fields:
      - `status` = `rejected`
      - `message` = `Posts were reviewed and rejected by the user. No content was published.`
    - Connect from the **false** output of `Approved?` to `Rejected – End`

15. **Configure credentials**
    - **Google Gemini(PaLM) Api account**
      - Required for both Gemini nodes
      - Ensure the API key/project has access to the chosen Gemini model
    - **X account OAuth2**
      - Required for `Post to X`
      - Verify write permissions for posting
    - **LinkedIn account OAuth2**
      - Required for `Post to Linkedin`
      - Verify posting permissions and correct actor/person context

16. **Optionally add documentation sticky notes**
    - Add sticky notes around each logical region:
      - Input
      - URL parsing
      - Reddit fetch
      - content extraction
      - summary
      - social generation
      - parsing
      - review
      - resolution
      - approval
      - publishing
      - rejection

17. **Test with a known public Reddit thread**
    - Submit a standard URL like:
      `https://www.reddit.com/r/<subreddit>/comments/<postId>/<slug>/`
    - Verify:
      - URL parsing works
      - Reddit fetch returns valid JSON
      - Gemini generates summary and JSON social output
      - approval form displays both drafts
      - approve path posts successfully
      - reject path ends with rejection status

18. **Recommended hardening before production**
    - Expand URL parser if you truly need `redd.it` short links
    - Add retry/error handling around Reddit and Gemini nodes
    - Validate LinkedIn person/author ID explicitly
    - Consider content moderation/review rules before publishing
    - Add logging or persistence for approved/rejected decisions

### Sub-workflow setup
This workflow does **not** invoke any sub-workflows and has only one entry point: the `Reddit URL Input` form trigger.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Converts any Reddit thread into ready-to-publish posts for X (Twitter) and LinkedIn, with a human review gate before publication. | Overall workflow purpose |
| Data flow: Reddit URL → Parse → Fetch API → Extract Content → AI Summarize → Generate Posts → Human Approval → Publish | Overall workflow structure |
| Key features: accepts Reddit thread URLs, extracts nested comments sorted by score, summarizes with Gemini, generates platform-specific drafts, includes human approval and rejection path. | Overall workflow capabilities |
| Tools used: Google Gemini 3.1 Flash Lite, Reddit JSON API, X (Twitter) API, LinkedIn API. | Technology stack |
| Important implementation note: the URL parsing code claims to support `redd.it` short links, but the current regex only matches full `reddit.com/r/.../comments/...` URLs. | Practical correction for maintainers |
| Important implementation note: the comment extractor recursively traverses available replies, but does not expand Reddit `more` placeholders, so some comments may remain unextracted on large threads. | Data completeness limitation |
| Important implementation note: both `Resolve Final Posts` and `Approved?` rely on hard-coded node names in expressions. Renaming those nodes requires updating the expressions too. | Maintenance consideration |