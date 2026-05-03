Generate book summaries with GPT-4o, Google APIs & Amazon Affiliate Links

https://n8nworkflows.xyz/workflows/generate-book-summaries-with-gpt-4o--google-apis---amazon-affiliate-links-11347


# Generate book summaries with GPT-4o, Google APIs & Amazon Affiliate Links

## 1. Workflow Overview

**Purpose:**  
This workflow collects a user’s requested book title (and optional author) via an n8n Form, enriches the request with **Google Books** metadata and **Google Custom Search** review snippets, asks **GPT-4o** (via the LangChain Agent node) to produce a **structured JSON** summary + recommendations, then **emails** an HTML report containing **Amazon affiliate links**, and finally **logs** the interaction to **Google Sheets**.

**Target use cases:**
- Automated “book advisor” for readers (Japanese-language email output)
- Lead magnet / newsletter automation that includes affiliate link monetization
- Internal library/research assistant for quick book briefs and similar reads

### 1.1 Input Reception & Configuration
Receives form input and centralizes API endpoints, keys, sheet ID, and Amazon associate tag.

### 1.2 Data Enrichment (Book metadata + Review context)
Looks up the book in Google Books; if found, extracts core metadata and retrieves review snippets from Google Custom Search.

### 1.3 AI Analysis (Structured)
Sends book details + review snippets to GPT-4o and enforces a JSON output schema (review summary + 3 related books).

### 1.4 Delivery & Logging
Builds a rich HTML email (cover image, summary, related books with affiliate links), sends it via Gmail, and appends a row to Google Sheets. If book is not found, sends an apology email instead.

---

## 2. Block-by-Block Analysis

### Block 1.1 — Input Reception & Configuration

**Overview:**  
Collects user input via an n8n form and sets configuration variables (API URLs, API keys, destination email, sheet ID, and Amazon affiliate tag) used throughout downstream nodes.

**Nodes Involved:**
- Sticky Note Main (documentation)
- Sticky Note Input (section label)
- **Book Input Form**
- **Workflow Configuration**

#### Node: Sticky Note Main
- **Type / Role:** Sticky Note (documentation only)
- **Configuration:** Contains purpose + setup steps (Google Sheets headers, credentials, config section guidance)
- **Connections:** None
- **Edge cases:** None (non-executing)

#### Node: Sticky Note Input
- **Type / Role:** Sticky Note (documentation only)
- **Connections:** None

#### Node: Book Input Form
- **Type / Role:** `formTrigger` — Entry point (web form)
- **Key configuration choices:**
  - Form title: “📚 Book Reading Advisor”
  - Button label: “Start Research”
  - Fields:
    - “書名 (Book Title)” (required)
    - “著者名 (Author Name - Optional)” (optional)
    - “メール送信先 (Email Address)” (required, email type)
- **Input/Output:**
  - **Output →** Workflow Configuration
- **Potential failures / edge cases:**
  - Users may enter variants/spelling differences; affects search accuracy.
  - Field names are Japanese/English mixed; downstream mappings must match exactly.

#### Node: Workflow Configuration
- **Type / Role:** `set` — Central config + normalization
- **Configuration choices (interpreted):**
  - Defines constants:
    - `googleBooksApiUrl` = `https://www.googleapis.com/books/v1/volumes`
    - `googleCustomSearchApiUrl` = `https://www.googleapis.com/customsearch/v1`
  - Stores placeholders (must be replaced):
    - `googleCustomSearchApiKey`
    - `googleCustomSearchEngineId`
    - `sheetId`
    - `amazonAffiliateTag`
  - Extracts user email from form:
    - `userEmail` = `{{ $json['メール送信先 (Email Address)'] }}`
  - `includeOtherFields: true` (keeps original form payload)
- **Input/Output:**
  - **Input ←** Book Input Form
  - **Output →** Search Book on Google Books API
- **Potential failures / edge cases:**
  - Placeholder values left unchanged will break Custom Search and/or affiliate tag behavior.
  - Note: form fields for title/author are not explicitly mapped here; downstream nodes must reference the form’s exact field names or mapped aliases (see “Search Book…” node for possible mismatch risk).

---

### Block 1.2 — Data Enrichment (Google Books + Google Custom Search)

**Overview:**  
Searches Google Books for a matching volume, branches if none are found, otherwise extracts metadata and searches the web for review snippets.

**Nodes Involved:**
- Sticky Note Enrichment (section label)
- **Search Book on Google Books API**
- **Check if Book Found**
- **Extract Book Details**
- **Search Book Reviews on Google**
- **Prepare AI Input**
- **Send Not Found Email** (error branch)

#### Node: Sticky Note Enrichment
- **Type / Role:** Sticky Note (documentation only)

#### Node: Search Book on Google Books API
- **Type / Role:** `httpRequest` — Calls Google Books REST API
- **Configuration choices:**
  - URL is constructed from config + query parameters:
    - Base: `{{ $('Workflow Configuration').first().json.googleBooksApiUrl }}`
    - Query:
      - `q=intitle:{title}+inauthor:{author?}`
      - `maxResults=1`
      - `langRestrict=ja`
  - Response format: JSON
- **Key expressions / variables:**
  - Uses `encodeURIComponent(...)` for query safety
  - References `{{$json.bookTitle}}` and `{{$json.authorName}}`
- **Input/Output:**
  - **Input ←** Workflow Configuration
  - **Output →** Check if Book Found
- **Potential failures / edge cases:**
  - **High-risk mapping mismatch:** The form fields are labeled “書名 (Book Title)” and “著者名 (Author Name - Optional)”, but this node expects `bookTitle` and `authorName`. If those properties are not present, the query becomes `intitle:undefined` and results will likely be empty.
    - Mitigation: add a Set node mapping form labels to `bookTitle`/`authorName`, or adjust expressions to `$json['書名 (Book Title)']`.
  - Google Books may return `totalItems=0` even for valid books (localization, title variants).
  - Rate limits / transient 429/5xx errors.

#### Node: Check if Book Found
- **Type / Role:** `if` — Branching based on `totalItems`
- **Condition:**
  - `{{ $json.totalItems }} > 0`
- **Input/Output:**
  - **Input ←** Search Book on Google Books API
  - **True →** Extract Book Details
  - **False →** Send Not Found Email
- **Potential failures / edge cases:**
  - If API returns unexpected structure (no `totalItems`), condition may fail (loose validation helps but still risky).
  - If `totalItems>0` but `items` missing, downstream extraction can error.

#### Node: Extract Book Details
- **Type / Role:** `set` — Normalizes metadata from Google Books response
- **Fields produced:**
  - `bookTitle` = `items[0].volumeInfo.title`
  - `bookAuthors` = join authors or “Unknown”
  - `bookDescription` = description or fallback
  - `bookIsbn`:
    - Prefer `ISBN_13` if present, else first identifier, else `N/A`
  - `bookCoverImage` = thumbnail or empty string
  - `bookPublisher`, `bookPublishedDate` with fallbacks
  - Keeps other fields (`includeOtherFields: true`)
- **Input/Output:**
  - **Input ←** Check if Book Found (true branch)
  - **Output →** Search Book Reviews on Google
- **Potential failures / edge cases:**
  - `items[0]` undefined if API response is malformed despite `totalItems>0`.
  - Some books have no `industryIdentifiers` or `imageLinks`; handled with optional chaining/fallbacks.

#### Node: Search Book Reviews on Google
- **Type / Role:** `httpRequest` — Calls Google Custom Search API
- **Configuration choices:**
  - URL includes:
    - `key` from config
    - `cx` (Search Engine ID) from config
    - `q` = `bookTitle + ' 書評 感想'` (Japanese “review/impressions”)
    - `num=5`
  - Response format: JSON
- **Input/Output:**
  - **Input ←** Extract Book Details
  - **Output →** Prepare AI Input
- **Potential failures / edge cases:**
  - Invalid/placeholder API key or engine ID → 403/400.
  - Custom Search often requires billing; may fail if not enabled.
  - `items` may be missing (no results), handled in next node.

#### Node: Prepare AI Input
- **Type / Role:** `set` — Consolidates review snippets into a single text blob
- **Key field:**
  - `reviewSnippets` = join `items[].snippet` with blank lines; fallback “No reviews found”
  - Keeps other fields
- **Input/Output:**
  - **Input ←** Search Book Reviews on Google
  - **Output →** Book Curator AI Agent
- **Potential failures / edge cases:**
  - If Custom Search returns a different structure, mapping may fail (but includes fallback if `items` falsy).

#### Node: Send Not Found Email
- **Type / Role:** `gmail` — Sends a plain text error email
- **When used:** Branch from Check if Book Found = false
- **Configuration:**
  - `sendTo` = `{{ $('Workflow Configuration').first().json.userEmail }}`
  - Subject: “📚 読書アドバイザー: 書籍が見つかりませんでした”
  - Email type: text
  - Message references: `{{ $('Workflow Configuration').first().json.bookTitle }}`
- **Input/Output:**
  - **Input ←** Check if Book Found (false branch)
  - **No further outputs**
- **Potential failures / edge cases:**
  - **Bug risk:** `Workflow Configuration` does not define `bookTitle`. If the intention was the user-entered title, this expression will resolve empty/undefined. Use the form field value or map it into config.
  - Gmail OAuth issues, sending limits, invalid recipient email.

---

### Block 1.3 — AI Analysis (GPT-4o + Structured Output)

**Overview:**  
Uses a LangChain Agent with GPT-4o to generate a Japanese summary (≤150 chars) and 3 related books, strictly formatted as JSON via a structured output parser.

**Nodes Involved:**
- Sticky Note AI (section label)
- **Book Curator AI Agent**
- **OpenAI Chat Model**
- **Structured Output Parser**

#### Node: Sticky Note AI
- **Type / Role:** Sticky Note (documentation only)

#### Node: OpenAI Chat Model
- **Type / Role:** `lmChatOpenAi` — Provides GPT-4o chat model to the agent
- **Configuration:**
  - Model: `gpt-4o`
  - Credentials: OpenAI API credential
- **Connections:**
  - **Output (ai_languageModel) →** Book Curator AI Agent
- **Potential failures / edge cases:**
  - Invalid API key / quota exceeded.
  - Model availability changes; ensure n8n/OpenAI node supports `gpt-4o`.
  - Latency/timeouts on large prompts (description + snippets).

#### Node: Structured Output Parser
- **Type / Role:** `outputParserStructured` — Enforces JSON schema-like structure
- **Configuration:**
  - Provides a schema example requiring:
    - `review_summary` (string)
    - `related_books` (array of objects with `title`, `author`, `reason`)
- **Connections:**
  - **Output (ai_outputParser) →** Book Curator AI Agent
- **Potential failures / edge cases:**
  - If the model returns invalid JSON or missing keys, parsing may fail and stop the workflow.
  - Japanese text inside JSON is fine, but must remain valid JSON escaping.

#### Node: Book Curator AI Agent
- **Type / Role:** `@n8n/n8n-nodes-langchain.agent` — Main LLM reasoning step
- **Prompt content (user message):**
  - Includes:
    - `bookTitle`, `bookAuthors`, `bookDescription`, `reviewSnippets`
- **System message (Japanese):**
  - Role: professional book curator
  - Tasks:
    1. Explain appeal within 150 characters
    2. Recommend 3 related books
  - Output must be JSON
- **Critical configuration:**
  - `promptType: define`
  - `hasOutputParser: true` (paired with Structured Output Parser)
- **Input/Output:**
  - **Input ←** Prepare AI Input
  - **Main output →** Build HTML Email
  - Also depends on:
    - **Language model input** from OpenAI Chat Model connection
    - **Output parser input** from Structured Output Parser connection
- **Potential failures / edge cases:**
  - Prompt variables missing if earlier nodes didn’t produce `bookTitle` etc. (mapping mismatch upstream can cascade).
  - Output may be valid JSON but not match expected keys; downstream code expects `review_summary` and `related_books`.

---

### Block 1.4 — Delivery & Logging

**Overview:**  
Generates a styled HTML email including the cover image, AI summary, and affiliate purchase links; sends via Gmail; logs the request and response to Google Sheets.

**Nodes Involved:**
- Sticky Note Delivery (section label)
- **Build HTML Email**
- **Send Email Notification**
- **Log to Google Sheets**

#### Node: Sticky Note Delivery
- **Type / Role:** Sticky Note (documentation only)

#### Node: Build HTML Email
- **Type / Role:** `code` — Constructs the final HTML and email subject
- **Key behaviors:**
  - Reads:
    - Affiliate tag from `Workflow Configuration` (`amazonAffiliateTag`)
    - Book details from `Extract Book Details`
    - AI output from the incoming item (`review_summary`, `related_books`)
  - Generates links:
    - Main Amazon search link uses ISBN if available, else title+author
    - Appends `&tag={affiliateTag}` if tag is not empty and not placeholder
    - Also builds Rakuten link but **does not output it in HTML** (computed but unused)
  - Produces output JSON:
    - `htmlEmail`
    - `emailSubject` = `📚 書籍レポート: {bookTitle}`
- **Input/Output:**
  - **Input ←** Book Curator AI Agent
  - **Output →** Send Email Notification
- **Potential failures / edge cases:**
  - If AI output parser fails upstream, this node won’t run.
  - If `related_books` is not an array, `.forEach` may throw.
  - External calls are not made here, but large HTML could hit email size limits if expanded.
  - Uses cross-node data (`$('Extract Book Details').first()`); if multiple items are processed concurrently, relying on `.first()` can mix data between runs in batch scenarios.

#### Node: Send Email Notification
- **Type / Role:** `gmail` — Sends the HTML report email
- **Configuration:**
  - `sendTo` = `{{ $('Workflow Configuration').first().json.userEmail }}`
  - `subject` = `{{ $json.emailSubject }}`
  - `message` = `{{ $json.htmlEmail }}`
  - (No explicit “emailType” set; Gmail node typically sends HTML when message contains HTML, but behavior can vary by node/version.)
- **Input/Output:**
  - **Input ←** Build HTML Email
  - **Output →** Log to Google Sheets
- **Potential failures / edge cases:**
  - OAuth token expired/revoked.
  - Recipient address invalid.
  - Gmail sending quotas.

#### Node: Log to Google Sheets
- **Type / Role:** `googleSheets` — Append/update history row
- **Operation:** `appendOrUpdate`
- **Target:**
  - Spreadsheet ID = `{{ $('Workflow Configuration').first().json.sheetId }}`
  - Sheet name = `History`
- **Column mapping (must exist as headers):**
  - `date` = `{{ $now.format('yyyy-MM-dd HH:mm') }}`
  - `book_title` = `{{ $('Extract Book Details').first().json.bookTitle }}`
  - `author` = `{{ $('Extract Book Details').first().json.bookAuthors }}`
  - `ai_comment` = `{{ $('Book Curator AI Agent').first().json.review_summary }}`
  - `user_email` = `{{ $('Workflow Configuration').first().json.userEmail }}`
- **Input/Output:**
  - **Input ←** Send Email Notification
  - **No further outputs**
- **Potential failures / edge cases:**
  - Sheet headers mismatch (must match exactly: `date`, `book_title`, `author`, `ai_comment`, `user_email`).
  - `appendOrUpdate` requires a key/matching strategy in some setups; if not configured, it may behave like append or error depending on n8n version and node settings.
  - Spreadsheet permissions / OAuth scope issues.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note Main | stickyNote | Workflow description & setup notes |  |  | # AI Book Advisor with Affiliate Support… (includes setup steps for Sheets + credentials + config) |
| Sticky Note Input | stickyNote | Section label |  |  | ## 1. Input & Config Accepts user requests via Form and defines API keys/tags. |
| Sticky Note Enrichment | stickyNote | Section label |  |  | ## 2. Data Enrichment Retrieves book details (Google Books) and search results (Custom Search) for context. |
| Sticky Note AI | stickyNote | Section label |  |  | ## 3. AI Analysis GPT-4o summarizes reviews and recommends related books in JSON format. |
| Sticky Note Delivery | stickyNote | Section label |  |  | ## 4. Delivery & Log Builds an HTML email with affiliate links, sends it, and saves data to Sheets. |
| Book Input Form | formTrigger | Entry point (collect title/author/email) | — | Workflow Configuration | ## 1. Input & Config… |
| Workflow Configuration | set | Store API URLs/keys, sheet ID, affiliate tag, user email | Book Input Form | Search Book on Google Books API | ## 1. Input & Config… |
| Search Book on Google Books API | httpRequest | Fetch volume metadata | Workflow Configuration | Check if Book Found | ## 2. Data Enrichment… |
| Check if Book Found | if | Branch on `totalItems > 0` | Search Book on Google Books API | Extract Book Details; Send Not Found Email | ## 2. Data Enrichment… |
| Extract Book Details | set | Normalize title/authors/ISBN/cover/description | Check if Book Found (true) | Search Book Reviews on Google | ## 2. Data Enrichment… |
| Search Book Reviews on Google | httpRequest | Fetch review snippets via Custom Search | Extract Book Details | Prepare AI Input | ## 2. Data Enrichment… |
| Prepare AI Input | set | Join snippets into `reviewSnippets` | Search Book Reviews on Google | Book Curator AI Agent | ## 2. Data Enrichment… |
| OpenAI Chat Model | lmChatOpenAi | Provide GPT-4o model | — (model provider) | Book Curator AI Agent (ai_languageModel) | ## 3. AI Analysis… |
| Structured Output Parser | outputParserStructured | Enforce JSON output structure | — (parser provider) | Book Curator AI Agent (ai_outputParser) | ## 3. AI Analysis… |
| Book Curator AI Agent | langchain agent | Generate JSON summary + recommendations | Prepare AI Input (+ model + parser) | Build HTML Email | ## 3. AI Analysis… |
| Build HTML Email | code | Render styled HTML + affiliate links | Book Curator AI Agent | Send Email Notification | ## 4. Delivery & Log… |
| Send Email Notification | gmail | Send HTML email to user | Build HTML Email | Log to Google Sheets | ## 4. Delivery & Log… |
| Log to Google Sheets | googleSheets | Append/update history row | Send Email Notification | — | # AI Book Advisor with Affiliate Support… (setup mentions required headers) |
| Send Not Found Email | gmail | Notify user book wasn’t found | Check if Book Found (false) | — | ## 2. Data Enrichment… |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n. Name it as desired (e.g., “Generate book summaries with GPT-4o, Google APIs & Amazon Affiliate Links”).

2. **Add a Form Trigger node**  
   - Node type: **Form Trigger**  
   - Title: `📚 Book Reading Advisor`  
   - Description: `Enter a book title. AI will research it and send you a summary with recommendations.`  
   - Button label: `Start Research`  
   - Fields:
     - Text (required): `書名 (Book Title)`
     - Text (optional): `著者名 (Author Name - Optional)`
     - Email (required): `メール送信先 (Email Address)`

3. **Add a Set node** named **Workflow Configuration**  
   - Node type: **Set**
   - Enable **Include Other Fields**
   - Add fields:
     - `googleBooksApiUrl` = `https://www.googleapis.com/books/v1/volumes`
     - `googleCustomSearchApiUrl` = `https://www.googleapis.com/customsearch/v1`
     - `googleCustomSearchApiKey` = your API key
     - `googleCustomSearchEngineId` = your CX (search engine ID)
     - `sheetId` = your Google Sheet ID
     - `amazonAffiliateTag` = your Amazon Associate tag (without `&tag=`; just the tag value)
     - `userEmail` = expression: `{{ $json['メール送信先 (Email Address)'] }}`
   - Connect: **Form Trigger → Workflow Configuration**

4. **(Recommended fix) Add a Set node to map form fields** (prevents `undefined` searches)  
   - Name: `Map Form Fields`
   - Fields:
     - `bookTitle` = `{{ $json['書名 (Book Title)'] }}`
     - `authorName` = `{{ $json['著者名 (Author Name - Optional)'] }}`  
   - Enable “Include Other Fields”
   - Connect: **Workflow Configuration → Map Form Fields**

5. **Add an HTTP Request node** named **Search Book on Google Books API**  
   - Method: GET
   - Response: JSON
   - URL expression (use Map Form Fields as input if you added it; otherwise ensure `bookTitle/authorName` exist):
     - Base: `{{ $('Workflow Configuration').first().json.googleBooksApiUrl }}`
     - Full URL template:  
       `={{ $('Workflow Configuration').first().json.googleBooksApiUrl }}?q=intitle:{{ encodeURIComponent($json.bookTitle) }}{{ $json.authorName ? '+inauthor:' + encodeURIComponent($json.authorName) : '' }}&maxResults=1&langRestrict=ja`
   - Connect: **Map Form Fields (or Workflow Configuration) → Search Book…**

6. **Add an IF node** named **Check if Book Found**  
   - Condition: Number `{{ $json.totalItems }}` **greater than** `0`
   - Connect: **Search Book… → Check if Book Found**

7. **Add a Set node** named **Extract Book Details** (true branch)  
   - Enable “Include Other Fields”
   - Add fields with expressions:
     - `bookTitle` = `{{ $json.items[0].volumeInfo.title }}`
     - `bookAuthors` = `{{ $json.items[0].volumeInfo.authors ? $json.items[0].volumeInfo.authors.join(', ') : 'Unknown' }}`
     - `bookDescription` = `{{ $json.items[0].volumeInfo.description || 'No description available' }}`
     - `bookIsbn` = `{{ $json.items[0].volumeInfo.industryIdentifiers ? $json.items[0].volumeInfo.industryIdentifiers.find(id => id.type === 'ISBN_13')?.identifier || $json.items[0].volumeInfo.industryIdentifiers[0]?.identifier : 'N/A' }}`
     - `bookCoverImage` = `{{ $json.items[0].volumeInfo.imageLinks?.thumbnail || '' }}`
     - `bookPublisher` = `{{ $json.items[0].volumeInfo.publisher || 'Unknown' }}`
     - `bookPublishedDate` = `{{ $json.items[0].volumeInfo.publishedDate || 'Unknown' }}`
   - Connect: **Check if Book Found (true) → Extract Book Details**

8. **Add an HTTP Request node** named **Search Book Reviews on Google**  
   - Method: GET
   - Response: JSON
   - URL expression:  
     `={{ $('Workflow Configuration').first().json.googleCustomSearchApiUrl }}?key={{ $('Workflow Configuration').first().json.googleCustomSearchApiKey }}&cx={{ $('Workflow Configuration').first().json.googleCustomSearchEngineId }}&q={{ encodeURIComponent($json.bookTitle + ' 書評 感想') }}&num=5`
   - Connect: **Extract Book Details → Search Book Reviews on Google**

9. **Add a Set node** named **Prepare AI Input**  
   - Enable “Include Other Fields”
   - Field:
     - `reviewSnippets` = `{{ $json.items ? $json.items.map(item => item.snippet).join('\n\n') : 'No reviews found' }}`
   - Connect: **Search Book Reviews on Google → Prepare AI Input**

10. **Add the OpenAI model node** named **OpenAI Chat Model**  
   - Node type: **OpenAI Chat Model** (LangChain)
   - Model: `gpt-4o`
   - Credentials: configure **OpenAI API** credential in n8n

11. **Add a Structured Output Parser node** named **Structured Output Parser**  
   - Provide a JSON example schema:
     - `review_summary` (string)
     - `related_books` array of `{ title, author, reason }`

12. **Add a LangChain Agent node** named **Book Curator AI Agent**  
   - Prompt type: “Define”
   - System message (Japanese; same intent):
     - Instruct it to be a professional book curator
     - Produce:
       1) 150-char summary
       2) 3 related books
     - Output **must** be JSON
   - User message/body expression (uses fields from previous nodes):
     - Includes `bookTitle`, `bookAuthors`, `bookDescription`, `reviewSnippets`
   - Connect:
     - **Prepare AI Input → Book Curator AI Agent (main)**
     - **OpenAI Chat Model → Book Curator AI Agent (ai_languageModel)**
     - **Structured Output Parser → Book Curator AI Agent (ai_outputParser)**

13. **Add a Code node** named **Build HTML Email**  
   - Paste the JavaScript that:
     - Pulls affiliate tag from `Workflow Configuration`
     - Pulls book details from `Extract Book Details`
     - Uses AI output `review_summary` and `related_books`
     - Builds HTML content and returns `{ htmlEmail, emailSubject }`
   - Connect: **Book Curator AI Agent → Build HTML Email**

14. **Add a Gmail node** named **Send Email Notification**  
   - Operation: Send
   - To: `{{ $('Workflow Configuration').first().json.userEmail }}`
   - Subject: `{{ $json.emailSubject }}`
   - Message: `{{ $json.htmlEmail }}`
   - Credentials: configure **Gmail OAuth2** in n8n
   - Connect: **Build HTML Email → Send Email Notification**

15. **Add a Google Sheets node** named **Log to Google Sheets**  
   - Operation: `appendOrUpdate`
   - Document ID: `{{ $('Workflow Configuration').first().json.sheetId }}`
   - Sheet: `History`
   - Map columns (must exist as headers):
     - `date` = `{{ $now.format('yyyy-MM-dd HH:mm') }}`
     - `book_title` = `{{ $('Extract Book Details').first().json.bookTitle }}`
     - `author` = `{{ $('Extract Book Details').first().json.bookAuthors }}`
     - `ai_comment` = `{{ $('Book Curator AI Agent').first().json.review_summary }}`
     - `user_email` = `{{ $('Workflow Configuration').first().json.userEmail }}`
   - Credentials: configure **Google Sheets OAuth2**
   - Connect: **Send Email Notification → Log to Google Sheets**

16. **Add a Gmail node** named **Send Not Found Email** (false branch)  
   - To: `{{ $('Workflow Configuration').first().json.userEmail }}`
   - Subject: `📚 読書アドバイザー: 書籍が見つかりませんでした`
   - Email type: text
   - Message: apology + include the requested title  
   - Connect: **Check if Book Found (false) → Send Not Found Email**
   - Recommended fix: reference the original form title field (or mapped `bookTitle`) rather than `Workflow Configuration.bookTitle`.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Create a Google Sheet with headers: `date`, `book_title`, `author`, `ai_comment`, `user_email`. | Mentioned in “AI Book Advisor with Affiliate Support” sticky note. |
| Configure credentials: OpenAI, Google Custom Search, Gmail. | Mentioned in “AI Book Advisor with Affiliate Support” sticky note. |
| Enter API keys and Amazon affiliate tag in the “1. Input & Config” section. | Mentioned in “AI Book Advisor with Affiliate Support” sticky note. |
| Disclaimer (French): content originates exclusively from an n8n automated workflow; compliant and uses legal/public data. | Provided by user in the prompt context. |