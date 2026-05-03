Score WhatsApp PDF resumes with OpenAI GPT-4o-mini and Supabase

https://n8nworkflows.xyz/workflows/score-whatsapp-pdf-resumes-with-openai-gpt-4o-mini-and-supabase-14814


# Score WhatsApp PDF resumes with OpenAI GPT-4o-mini and Supabase

The user has provided a complete n8n workflow JSON for a "Score WhatsApp PDF resumes with OpenAI GPT-4o-mini and Supabase" workflow. They want me to produce a comprehensive, structured reference document following the specified format: Workflow Overview, Block-by-Block Analysis, Summary Table, Reproducing the Workflow from Scratch, General Notes & Resources.

Let me analyze the workflow thoroughly.

## Nodes in the workflow:

1. **Main Sticky** - Sticky note (color 2)
2. **WhatsApp Trigger** - WhatsApp trigger node (listens for incoming messages)
3. **Conversation State Manager** - Code node (manages conversation state via Supabase)
4. **Route Action** - Switch node (routes based on action)
5. **Send Conversation Reply** - WhatsApp send node
6. **Send Processing Message** - WhatsApp send node
7. **Get Resume URL** - WhatsApp media URL get node
8. **Download Resume** - HTTP Request node (downloads PDF)
9. **Extract Resume Text** - Extract from File node (PDF extraction)
10. **Set Resume Data** - Set node (organizes resume text, target role, from phone)
11. **Scoring Engine** - Code node (calculates score)
12. **OpenAI - Rejection Analysis** - HTTP Request node (calls OpenAI API)
13. **Parse AI Response** - Set node (parses JSON response)
14. **Merge Score + AI** - Merge node (combines scoring and AI results)
15. **Format Result Message** - Code node (formats final WhatsApp message)
16. **Send Final Result** - WhatsApp send node
17. **Upload Resume to Storage** - Code node (uploads PDF to Supabase Storage)
18. **Section 1** - Sticky note (Trigger & State)
19. **Section 2** - Sticky note (Processing & Analysis)

## Connections:

- WhatsApp Trigger → Conversation State Manager
- Conversation State Manager → Route Action
- Route Action output 0 (Process Resume) → Send Processing Message
- Route Action output 1 (Send Reply) → Send Conversation Reply
- Send Processing Message → Get Resume URL
- Get Resume URL → Download Resume
- Download Resume → Extract Resume Text AND Upload Resume to Storage
- Extract Resume Text → Set Resume Data
- Set Resume Data → Scoring Engine AND OpenAI - Rejection Analysis
- Scoring Engine → Merge Score + AI (input 0)
- OpenAI - Rejection Analysis → Parse AI Response
- Parse AI Response → Merge Score + AI (input 1)
- Merge Score + AI → Format Result Message
- Format Result Message → Send Final Result

## Sticky notes content:

- **Main Sticky**: "## WhatsApp Career Ranking Bot\nAnalyze resumes via WhatsApp to provide career scores, ATS feedback, and ranking insights.\n\n\n### How it works\n1. User triggers bot via keyword.\n\n2. Collects name, target role, and PDF resume.\n\n3. Scoring Engine calculates metrics.\n\n4. AI identifies rejection reasons.\n\n5. Returns personalized report to user."

- **Section 1**: "## 1. Trigger & State\nHandles WhatsApp webhook and manages user conversation progress in Supabase."

- **Section 2**: "## 2. Processing & Analysis\nDownloads resume, extracts text, performs scoring, and generates AI improvement suggestions."

The Section 1 sticky note visually covers: WhatsApp Trigger, Conversation State Manager, Route Action, Send Conversation Reply, Send Processing Message

The Section 2 sticky note visually covers: Get Resume URL, Download Resume, Extract Resume Text, Set Resume Data, Scoring Engine, OpenAI - Rejection Analysis, Parse AI Response, Upload Resume to Storage, Merge Score + AI, Format Result Message, Send Final Result

Now let me build the full documentation.

### 1. Workflow Overview

The workflow is a WhatsApp-based resume ranking bot. It receives WhatsApp messages, manages multi-turn conversation state in Supabase, collects user information (name, target role, PDF resume), downloads and extracts the PDF, runs a scoring engine, calls OpenAI GPT-4o-mini for rejection analysis, merges results, formats a report, and sends it back to the user.

Logical blocks:
1. **Trigger & State Management** (WhatsApp Trigger → Conversation State Manager → Route Action → Send Conversation Reply / Send Processing Message)
2. **Resume Retrieval & Storage** (Get Resume URL → Download Resume → Upload Resume to Storage)
3. **Resume Text Extraction** (Download Resume → Extract Resume Text → Set Resume Data)
4. **Scoring & AI Analysis** (Set Resume Data → Scoring Engine | OpenAI - Rejection Analysis → Parse AI Response)
5. **Result Merging & Formatting** (Merge Score + AI → Format Result Message → Send Final Result)

### 2. Block-by-Block Analysis

Let me detail each block.

#### Block 1: Trigger & State Management

**Overview**: Receives incoming WhatsApp messages, manages conversation state via Supabase, determines the next action based on user state and input, and routes to appropriate response handling.

**Nodes Involved**: WhatsApp Trigger, Conversation State Manager, Route Action, Send Conversation Reply, Send Processing Message

**Node Details**:

- **WhatsApp Trigger**: Type: `n8n-nodes-base.whatsAppTrigger`, version 1. Listens for incoming WhatsApp message webhooks. Subscribed to "messages" updates. No special configuration beyond the webhook ID. This is the entry point of the workflow. Requires WhatsApp Business Cloud API credentials configured in n8n.

- **Conversation State Manager**: Type: `n8n-nodes-base.code`, version 2. This is a comprehensive Code node that:
  - Reads incoming WhatsApp message payload
  - Filters out status messages and empty messages
  - Extracts sender phone number, text content, and document metadata
  - Loads existing session from Supabase `whatsapp_sessions` table using REST API (GET)
  - Implements a state machine with states: IDLE, WAITING_YES, WAITING_NAME, WAITING_ROLE, WAITING_RESUME, PROCESSING
  - Determines next action based on state and user input (keywords like "CHECK MY RANK", "YES", etc.)
  - Generates appropriate reply text
  - Upserts session back to Supabase (POST with on_conflict=phone)
  - Returns JSON with: action, replyText, from (phone), session object, documentId, messageType
  
  Key expressions:
  - `SUPABASE_URL` and `SUPABASE_KEY` are hardcoded (must be updated)
  - Multiple keyword variations accepted for triggers
  - PDF MIME type check for document uploads
  - Profile name extraction from contacts array

  Edge cases:
  - Supabase connection errors are caught and logged
  - Status messages are ignored
  - Invalid or empty messages return IGNORE action
  - Session state defaults to IDLE if not found

- **Route Action**: Type: `n8n-nodes-base.switch`, version 3.2. Routes based on `action` field:
  - Output "Process Resume": when action equals "PROCESS_RESUME"
  - Output "Send Reply": when action is not PROCESS_RESUME and not IGNORE
  - Fallback: none (no output for IGNORE actions)
  
  Uses strict string comparison, case-sensitive.

- **Send Conversation Reply**: Type: `n8n-nodes-base.whatsApp`, version 1. Sends a text message back to the WhatsApp user. Parameters:
  - `operation`: send
  - `textBody`: expression `{{ $json.replyText }}`
  - `recipientPhoneNumber`: expression `{{ $json.from }}`
  - `phoneNumberId`: "YOUR_PHONE_NUMBER_HERE" (must be replaced)
  
  Requires WhatsApp Business API credentials.

- **Send Processing Message**: Type: `n8n-nodes-base.whatsApp`, version 1. Same configuration as Send Conversation Reply but triggered on the PROCESS_RESUME action path. Sends the "Analyzing..." message.

#### Block 2: Resume Retrieval & Storage

**Overview**: Retrieves the resume media URL from WhatsApp, downloads the PDF binary, and uploads it to Supabase Storage for persistence.

**Nodes Involved**: Get Resume URL, Download Resume, Upload Resume to Storage

**Node Details**:

- **Get Resume URL**: Type: `n8n-nodes-base.whatsApp`, version 1. Uses the WhatsApp media resource operation `mediaUrlGet` with the document ID from the previous Conversation State Manager output. Expression: `{{ $('Conversation State Manager').item.json.documentId }}`. Requires WhatsApp API credentials.

- **Download Resume**: Type: `n8n-nodes-base.httpRequest`, version 4.2. Downloads the PDF file using the URL from Get Resume URL. Configuration:
  - `url`: expression `{{ $json.url }}`
  - `responseFormat`: file (binary)
  - `authentication`: predefinedCredentialType (whatsAppApi)
  
  Output includes binary data of the PDF file.

- **Upload Resume to Storage**: Type: `n8n-nodes-base.code`, version 2. Code node that:
  - Reads the binary PDF buffer using `this.helpers.getBinaryDataBuffer`
  - Generates a unique filename using phone number and timestamp
  - Uploads to Supabase Storage bucket "resumes" via POST to `/storage/v1/object/resumes/{filename}`
  - Sets Content-Type to application/pdf
  - Saves the public URL back to `whatsapp_sessions` table via PATCH
  - Handles cases where binary data may not be available
  - Logs detailed debug information
  
  Edge cases:
  - If no binary key found, returns empty resume_url
  - Upload errors are caught and logged
  - Optimistic URL construction even if upload response is ambiguous

#### Block 3: Resume Text Extraction

**Overview**: Extracts readable text from the downloaded PDF and prepares data for scoring and AI analysis.

**Nodes Involved**: Extract Resume Text, Set Resume Data

**Node Details**:

- **Extract Resume Text**: Type: `n8n-nodes-base.extractFromFile`, version 1. Operation: pdf. Extracts text content from the PDF binary received from Download Resume. No special options configured.

- **Set Resume Data**: Type: `n8n-nodes-base.set`, version 3.4. Assigns three fields:
  - `resume_text`: expression `{{ $json.text }}` (from Extract Resume Text output)
  - `target_role`: expression `{{ $('Conversation State Manager').item.json.session.target_role }}` (references session data)
  - `from`: expression `{{ $('Conversation State Manager').item.json.from }}` (sender phone number)

#### Block 4: Scoring & AI Analysis

**Overview**: Runs two parallel processes — a deterministic scoring engine that calculates a career score across 5 dimensions, and an OpenAI GPT-4o-mini call that generates rejection reasons and improvement tips.

**Nodes Involved**: Scoring Engine, OpenAI - Rejection Analysis, Parse AI Response

**Node Details**:

- **Scoring Engine**: Type: `n8n-nodes-base.code`, version 2. Extensive Code node implementing:
  - **Impact Score (0-25)**: Counts unique impact verbs and quantitative metrics in resume text
  - **Relevance Score (0-25)**: Matches resume keywords against role-specific keyword maps for 17+ job roles
  - **ATS Readiness Score (0-20)**: Checks for common resume sections, email, phone, LinkedIn, GitHub, and special character density
  - **Clarity Score (0-15)**: Evaluates average sentence length, bullet point usage, and year range mentions
  - **Completeness Score (0-15)**: Assesses word count, education mentions, certifications, and project mentions
  - **Final Score**: Sum of all dimensions (0-100)
  - **Percentile calculation**: Queries Supabase `resume_scores` table for same target_role, ranks against other candidates
  - **Interview probability**: High/Moderate/Low/Very Low based on score thresholds
  - **Data extraction**: Email and LinkedIn URL from resume text
  - **Database updates**: Saves score to `resume_scores` table, updates email/LinkedIn/resume_url in `whatsapp_sessions`
  
  Expressions:
  - References `SUPABASE_URL` and `SUPABASE_KEY` (must be updated)
  - References `$input.first().json.resume_text`, `target_role`, `from`
  
  Edge cases:
  - Fallback to generic keywords if no role match found
  - Score capped at dimension maximums
  - Percentile defaults to "First submission!" with single candidate
  - Database errors caught and logged

- **OpenAI - Rejection Analysis**: Type: `n8n-nodes-base.httpRequest`, version 4.2. Makes a POST request to OpenAI API (or custom endpoint). Configuration:
  - `url`: "YOUR_API_ENDPOINT_HERE" (must be replaced with actual OpenAI API URL, likely `https://api.openai.com/v1/chat/completions`)
  - `method`: POST
  - `authentication`: predefinedCredentialType (openAiApi)
  - `jsonBody`: Expression constructing the request body with:
    - model: "gpt-4o-mini"
    - max_tokens: 800
    - response_format: { "type": "json_object" }
    - System message: "You are an expert ATS resume reviewer. Return ONLY valid JSON, no extra text."
    - User message: Includes target role and resume text (truncated to 3000 chars), requesting JSON with rejection_reasons, fixes, and one_liner
  - `sendBody`: true
  - `specifyBody`: json

  Requires OpenAI API credentials configured in n8n.

- **Parse AI Response**: Type: `n8n-nodes-base.set`, version 3.4. Parses the OpenAI response:
  - `ai_analysis`: expression `{{ JSON.parse($json.choices[0].message.content) }}` — extracts the JSON content from the chat completion response

#### Block 5: Result Merging & Formatting

**Overview**: Merges the deterministic scoring results with the AI-generated analysis, formats a comprehensive WhatsApp message, resets the user's session state, and sends the final report.

**Nodes Involved**: Merge Score + AI, Format Result Message, Send Final Result

**Node Details**:

- **Merge Score + AI**: Type: `n8n-nodes-base.merge`, version 3. Mode: combine, combineBy: combineAll. Merges the two input streams:
  - Input 0: From Scoring Engine (finalScore, percentile, etc.)
  - Input 1: From Parse AI Response (ai_analysis)
  - Produces a single item with all fields combined

- **Format Result Message**: Type: `n8n-nodes-base.code`, version 2. Code node that:
  - Builds a formatted WhatsApp message with:
    - Career Rank Report header
    - Target role
    - Score out of 100 with visual bar (█/░)
    - Percentile with emoji indicator
    - Rank out of total candidates
    - Interview probability
    - Top 3 rejection reasons from AI
    - One-liner summary
    - Top 3 fixes from AI
    - Call-to-action link to meracareer.io
  - Resets the user's session state to IDLE in Supabase via PATCH
  - Returns JSON with `from` and `message` fields
  
  Expressions reference `SUPABASE_URL` and `SUPABASE_KEY` (must be updated)

- **Send Final Result**: Type: `n8n-nodes-base.whatsApp`, version 1. Sends the formatted message to the user:
  - `textBody`: expression `{{ $json.message }}`
  - `recipientPhoneNumber`: expression `{{ $json.from }}`
  - `phoneNumberId`: "YOUR_PHONE_NUMBER_HERE" (must be replaced)

### 3. Summary Table

Let me compile all nodes with their details.

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Main Sticky | stickyNote | Visual documentation | — | — | "## WhatsApp Career Ranking Bot..." |
| WhatsApp Trigger | whatsAppTrigger | Entry point – receives incoming WhatsApp messages | — | Conversation State Manager | "## 1. Trigger & State..." |
| Conversation State Manager | code | Manages conversation state machine and Supabase session | WhatsApp Trigger | Route Action | "## 1. Trigger & State..." |
| Route Action | switch | Routes based on action: PROCESS_RESUME or other replies | Conversation State Manager | Send Processing Message, Send Conversation Reply | "## 1. Trigger & State..." |
| Send Conversation Reply | whatsApp | Sends text replies for non-processing actions | Route Action (Send Reply) | — | "## 1. Trigger & State..." |
| Send Processing Message | whatsApp | Sends "Analyzing..." message when resume is received | Route Action (Process Resume) | Get Resume URL | "## 1. Trigger & State..." |
| Get Resume URL | whatsApp | Retrieves WhatsApp media URL for the PDF document | Send Processing Message | Download Resume | "## 2. Processing & Analysis..." |
| Download Resume | httpRequest | Downloads the PDF binary from WhatsApp media server | Get Resume URL | Extract Resume Text, Upload Resume to Storage | "## 2. Processing & Analysis..." |
| Extract Resume Text | extractFromFile | Extracts text content from the PDF | Download Resume | Set Resume Data | "## 2. Processing & Analysis..." |
| Upload Resume to Storage | code | Uploads PDF binary to Supabase Storage and saves URL | Download Resume | — | "## 2. Processing & Analysis..." |
| Set Resume Data | set | Organizes resume text, target role, and sender phone for downstream | Extract Resume Text | Scoring Engine, OpenAI - Rejection Analysis | "## 2. Processing & Analysis..." |
| Scoring Engine | code | Calculates career score across 5 dimensions and percentile ranking | Set Resume Data | Merge Score + AI | "## 2. Processing & Analysis..." |
| OpenAI - Rejection Analysis | httpRequest | Calls OpenAI GPT-4o-mini for rejection reasons and improvement tips | Set Resume Data | Parse AI Response | "## 2. Processing & Analysis..." |
| Parse AI Response | set | Parses OpenAI JSON response into structured object | OpenAI - Rejection Analysis | Merge Score + AI | "## 2. Processing & Analysis..." |
| Merge Score + AI | merge | Combines scoring engine output and AI analysis into one item | Scoring Engine, Parse AI Response | Format Result Message | "## 2. Processing & Analysis..." |
| Format Result Message | code | Formats final WhatsApp report message and resets session | Merge Score + AI | Send Final Result | "## 2. Processing & Analysis..." |
| Send Final Result | whatsApp | Sends the final career rank report to the user | Format Result Message | — | "## 2. Processing & Analysis..." |
| Section 1 | stickyNote | Visual section label | — | — | "## 1. Trigger & State..." |
| Section 2 | stickyNote | Visual section label | — | — | "## 2. Processing & Analysis..." |

Now let me think about sticky note assignments more carefully:

- **Main Sticky** is a standalone sticky note at position [-1872, 64] with size 500x456. It covers the general area. It doesn't directly cover any specific node, but is a general description. I'll note it but won't assign it to specific nodes.

- **Section 1** sticky note is at [-1344, 64] with size 1276x456. Looking at node positions:
  - WhatsApp Trigger: [-1328, 256]
  - Conversation State Manager: [-1104, 256]
  - Route Action: [-880, 256]
  - Send Conversation Reply: [-656, 352]
  - Send Processing Message: [-656, 160]
  
  All these nodes fall within the Section 1 sticky area. So Section 1 note applies to all these nodes.

- **Section 2** sticky note is at [0, 64] with size 1516x456. Looking at node positions:
  - Get Resume URL: [-432, 160]
  - Download Resume: [-208, 160]
  - Extract Resume Text: [64, 160]
  - Upload Resume to Storage: [64, 336]
  - Set Resume Data: [288, 160]
  - Scoring Engine: [624, 160]
  - OpenAI - Rejection Analysis: [512, 256]
  - Parse AI Response: [736, 256]
  - Merge Score + AI: [928, 176]
  - Format Result Message: [1120, 176]
  - Send Final Result: [1312, 176]
  
  Wait, Get Resume URL is at [-432, 160] which is outside the Section 2 sticky note (which starts at x=0). But it's close. Let me check more carefully.

  Section 2: position [0, 64], width 1516, height 456. So x range: 0 to 1516, y range: 64 to 520.

  Get Resume URL at [-432, 160] is outside the x range. But it's part of the processing pipeline. Hmm, the sticky note may not perfectly cover all nodes due to n8n canvas positioning. I'll still apply Section 2 to all processing nodes since the description says "Downloads resume, extracts text, performs scoring, and generates AI improvement suggestions" which clearly covers them.

  Actually, looking more carefully:
  - Get Resume URL at x=-432 is before Section 2 starts (x=0), so it might not be covered. But it's part of the processing flow. The sticky note intent seems to cover the processing pipeline. I'll be precise about what the sticky notes say and which nodes they apply to based on their content descriptions.

Let me re-evaluate: Section 1 says "Handles WhatsApp webhook and manages user conversation progress in Supabase." This clearly applies to: WhatsApp Trigger, Conversation State Manager, Route Action, Send Conversation Reply, Send Processing Message.

Section 2 says "Downloads resume, extracts text, performs scoring, and generates AI improvement suggestions." This clearly applies to: Get Resume URL, Download Resume, Extract Resume Text, Upload Resume to Storage, Set Resume Data, Scoring Engine, OpenAI - Rejection Analysis, Parse AI Response, Merge Score + AI, Format Result Message, Send Final Result.

I'll assign sticky notes accordingly in the table.

### 4. Reproducing the Workflow from Scratch

I need to provide a step-by-step guide to rebuild this workflow manually in n8n.

Steps:
1. Create WhatsApp Trigger node
2. Create Conversation State Manager Code node
3. Create Route Action Switch node
4. Create Send Conversation Reply WhatsApp node
5. Create Send Processing Message WhatsApp node
6. Create Get Resume URL WhatsApp node
7. Create Download Resume HTTP Request node
8. Create Upload Resume to Storage Code node
9. Create Extract Resume Text node
10. Create Set Resume Data Set node
11. Create Scoring Engine Code node
12. Create OpenAI - Rejection Analysis HTTP Request node
13. Create Parse AI Response Set node
14. Create Merge Score + AI Merge node
15. Create Format Result Message Code node
16. Create Send Final Result WhatsApp node

Plus connections and credential setup.

Let me detail each step with configuration details.

### 5. General Notes & Resources

Include notes about:
- WhatsApp Business API requirements
- Supabase setup
- OpenAI API setup
- The meracareer.io link
- Customization options

Now let me write the full document.

I need to be thorough but not include raw JSON. I'll reference configuration choices without pasting the full code.

Let me think about edge cases and potential failures for each node:

- **WhatsApp Trigger**: Requires webhook URL configured in Meta Developer dashboard. Must subscribe to "messages" event. Phone Number ID must match. If Meta app is in sandbox mode, only whitelisted numbers can message.

- **Conversation State Manager**: Could fail if Supabase is unreachable. Hardcoded credentials need updating. Message parsing could fail on unexpected message types. State machine could get stuck if user sends unexpected input.

- **Route Action**: Strict string comparison means action values must exactly match what Conversation State Manager outputs.

- **WhatsApp Send nodes**: All require correct Phone Number ID. Messages over 4096 characters will fail. Rate limits apply.

- **Get Resume URL**: Requires valid documentId. Media may expire after a certain time.

- **Download Resume**: Could fail if media URL is expired. Binary data could be large.

- **Upload Resume to Storage**: Supabase Storage bucket must exist. RLS policies must allow upload. File size limits apply.

- **Extract Resume Text**: Could fail on password-protected or image-only PDFs. Output quality depends on PDF structure.

- **Scoring Engine**: Complex logic with many regex patterns. Could produce unexpected scores on unusual resume formats. Supabase queries could fail. Role matching is approximate.

- **OpenAI - Rejection Analysis**: Could fail if API key is invalid or quota exceeded. Response format must be JSON. Truncating resume to 3000 chars could lose important content. Timeout possible on large requests.

- **Parse AI Response**: Could fail if OpenAI doesn't return valid JSON despite the JSON mode instruction.

- **Merge Score + AI**: Both inputs must be present; if either fails, merge will wait indefinitely or produce incomplete data.

- **Format Result Message**: Session reset could fail. Message could exceed WhatsApp character limits.

- **Send Final Result**: Same constraints as other WhatsApp send nodes.

Now let me write the complete document.## 1. Workflow Overview

**Purpose:**  
This workflow implements a WhatsApp-based resume ranking chatbot. A user sends a keyword (“CHECK MY RANK”) on WhatsApp, steps through a short multi-turn conversation (name → target job role → PDF upload), and receives a personalised career score, ATS readiness assessment, rejection reasons, and improvement tips — all within roughly 60 seconds.

**Target Use Cases:**  
Career coaches, job portals, HR-tech startups, or recruitment agencies that want to offer a zero-friction, self-serve resume evaluation tool inside WhatsApp.

**Logical Blocks:**

| # | Block | Description |
|---|-------|-------------|
| 1 | **Trigger & State Management** | Receives WhatsApp webhooks, manages per-user conversation state in Supabase, and routes the next action. |
| 2 | **Resume Retrieval & Storage** | Fetches the WhatsApp media URL for the uploaded PDF, downloads the binary, and persists it to Supabase Storage. |
| 3 | **Text Extraction & Data Preparation** | Extracts plain text from the PDF and bundles it with the user’s target role and phone number for downstream processing. |
| 4 | **Scoring & AI Analysis** | Runs a deterministic five-dimension scoring engine and, in parallel, calls OpenAI GPT‑4o‑mini for rejection reasons and fix suggestions. |
| 5 | **Result Merging, Formatting & Delivery** | Merges both analysis branches, formats a WhatsApp message report, resets the user session, and sends the final reply. |

---

## 2. Block-by-Block Analysis

### Block 1 — Trigger & State Management

**Overview:**  
This block is the entry point. It listens for incoming WhatsApp messages, loads or creates a Supabase session for the sender, runs a state machine that walks the user through name → role → resume, determines the next action, and routes the flow accordingly.

**Nodes Involved:**  
- WhatsApp Trigger  
- Conversation State Manager  
- Route Action  
- Send Conversation Reply  
- Send Processing Message  

---

#### WhatsApp Trigger

| Property | Detail |
|----------|--------|
| **Type** | `n8n-nodes-base.whatsAppTrigger` (v1) |
| **Technical Role** | Webhook entry point; receives message events from Meta’s WhatsApp Business API |
| **Configuration** | Subscribed to `messages` updates only; webhook ID auto-generated |
| **Credentials** | WhatsApp Business Cloud API (Meta App token + Phone Number ID) |
| **Output** | Full WhatsApp webhook payload (contacts, messages, etc.) |
| **Edge Cases** | • Webhook URL must be configured in Meta Developer dashboard under the app’s WhatsApp section. <br>• If the app is in Sandbox mode, only whitelisted numbers can message the bot. <br>• Status-type messages (delivered, read) are filtered out downstream. |

---

#### Conversation State Manager

| Property | Detail |
|----------|--------|
| **Type** | `n8n-nodes-base.code` (v2) |
| **Technical Role** | State machine orchestrator; reads/writes Supabase `whatsapp_sessions`, decides next action and reply text |
| **Key Logic** | <ul><li>Ignores status messages and empty payloads.</li><li>Extracts sender phone, text body, and document metadata.</li><li>Loads existing session from Supabase (GET `whatsapp_sessions?phone=eq.{from}`).</li><li>Defaults to `{state:'IDLE', name:'', target_role:''}` if no session exists.</li><li>Runs a six-state machine: `IDLE → WAITING_YES → WAITING_NAME → WAITING_ROLE → WAITING_RESUME → PROCESSING`.</li><li>Trigger keywords (case-insensitive, punctuation stripped): `CHECK MY RANK`, `CHECK`, `CHECKMYRANK`, `START`, `RANK`.</li><li>Acceptance words for YES step: `YES`, `Y`, `YEAH`, `YEP`, `SURE`, `OK`, `OKAY`.</li><li>Validates name length 2–100 chars, role length 2–150 chars.</li><li>Only accepts PDF MIME type (`application/pdf`) for resumes; rejects DOC/DOCX.</li><li>Upserts session after every state transition (POST with `on_conflict=phone`).</li></ul> |
| **Expressions / Variables** | • `SUPABASE_URL` – hardcoded, must be updated. <br>• `SUPABASE_KEY` – placeholder JWT, must be replaced. <br>• `$input.first().json` – incoming WhatsApp payload. |
| **Output Fields** | `action`, `replyText`, `from`, `session`, `documentId`, `messageType` |
| **Potential Failures** | • Supabase connectivity errors (caught and logged). <br>• Unexpected message types causing `msg` fields to be undefined (guarded by early checks). <br>• State gets stuck in PROCESSING if downstream fails (user would see “Still analyzing…”). |

---

#### Route Action

| Property | Detail |
|----------|--------|
| **Type** | `n8n-nodes-base.switch` (v3.2) |
| **Technical Role** | Two-output router based on the `action` field |
| **Rules** | <ul><li>**Output “Process Resume”**: `action` equals `PROCESS_RESUME`</li><li>**Output “Send Reply”**: `action` not equals `PROCESS_RESUME` **and** not equals `IGNORE`</li></ul> Fallback output is disabled (`none`). |
| **Case Sensitivity** | Strict (case-sensitive) string comparison |
| **Edge Cases** | • If `action` is `IGNORE`, no output is produced and the execution silently ends. |

---

#### Send Conversation Reply

| Property | Detail |
|----------|--------|
| **Type** | `n8n-nodes-base.whatsApp` (v1) |
| **Technical Role** | Sends a text reply for all non-processing actions (intro, help, ask name, ask role, ask resume, validation errors, etc.) |
| **Configuration** | <ul><li>`operation`: send</li><li>`textBody`: `{{ $json.replyText }}`</li><li>`recipientPhoneNumber`: `{{ $json.from }}`</li><li>`phoneNumberId`: `YOUR_PHONE_NUMBER_HERE` (must be replaced)</li></ul> |
| **Credentials** | WhatsApp Business Cloud API |
| **Edge Cases** | • WhatsApp message length limit (~4096 chars). <br>• Rate limiting by Meta. |

---

#### Send Processing Message

| Property | Detail |
|----------|--------|
| **Type** | `n8n-nodes-base.whatsApp` (v1) |
| **Technical Role** | Sends the “Analyzing… hang tight 🔍” message when a PDF resume is received |
| **Configuration** | Same pattern as Send Conversation Reply (uses `replyText` and `from` from the PROCESS_RESUME action) |
| **Edge Cases** | Same as Send Conversation Reply |

---

### Block 2 — Resume Retrieval & Storage

**Overview:**  
Once the user uploads a PDF, this block obtains the WhatsApp media URL, downloads the binary file, and stores it in a Supabase Storage bucket for archival.

**Nodes Involved:**  
- Get Resume URL  
- Download Resume  
- Upload Resume to Storage  

---

#### Get Resume URL

| Property | Detail |
|----------|--------|
| **Type** | `n8n-nodes-base.whatsApp` (v1) |
| **Technical Role** | Resolves the WhatsApp media ID to a downloadable URL |
| **Configuration** | <ul><li>`resource`: media</li><li>`operation`: mediaUrlGet</li><li>`mediaGetId`: `{{ $('Conversation State Manager').item.json.documentId }}`</li></ul> |
| **Input** | Output of Send Processing Message |
| **Output** | JSON containing `url` field |
| **Credentials** | WhatsApp Business Cloud API |
| **Edge Cases** | • Media ID may expire; URL is time-limited. |

---

#### Download Resume

| Property | Detail |
|----------|--------|
| **Type** | `n8n-nodes-base.httpRequest` (v4.2) |
| **Technical Role** | Downloads the PDF binary from the WhatsApp media server |
| **Configuration** | <ul><li>`url`: `{{ $json.url }}`</li><li>`response.responseFormat`: file</li><li>`authentication`: predefinedCredentialType → `whatsAppApi`</li></ul> |
| **Output** | Binary data (PDF) attached to the item |
| **Edge Cases** | • Network timeout on large files. <br>• Authentication failure if WhatsApp token is invalid. |

---

#### Upload Resume to Storage

| Property | Detail |
|----------|--------|
| **Type** | `n8n-nodes-base.code` (v2) |
| **Technical Role** | Uploads the raw PDF buffer to Supabase Storage and persists the public URL in the user’s session |
| **Key Logic** | <ul><li>Detects the binary key dynamically (`Object.keys(allBinary)[0]`).</li><li>Reads the buffer via `this.helpers.getBinaryDataBuffer(0, binaryKey)`.</li><li>Generates filename: `{sanitized_phone}_{timestamp}.pdf`.</li><li>POSTs to Supabase Storage `/storage/v1/object/resumes/{filename}` with `Content-Type: application/pdf` and `x-upsert: true`.</li><li>PATCHes `whatsapp_sessions` to store `resume_url`.</li></ul> |
| **Expressions / Variables** | `SUPABASE_URL`, `SUPABASE_KEY` (placeholders) |
| **Output** | `{ resume_url, phone }` |
| **Potential Failures** | • No binary data available (returns `resume_url: ''`). <br>• Supabase Storage bucket “resumes” must exist and allow service-role uploads. <br>• File size may exceed Supabase storage limits. |

---

### Block 3 — Text Extraction & Data Preparation

**Overview:**  
Extracts readable text from the PDF and assembles the data payload needed for scoring and AI analysis.

**Nodes Involved:**  
- Extract Resume Text  
- Set Resume Data  

---

#### Extract Resume Text

| Property | Detail |
|----------|--------|
| **Type** | `n8n-nodes-base.extractFromFile` (v1) |
| **Technical Role** | Parses the downloaded PDF into plain text |
| **Configuration** | `operation`: pdf; default options |
| **Input** | Binary PDF from Download Resume |
| **Output** | JSON with `text` field containing the extracted content |
| **Edge Cases** | • Scanned/image-only PDFs produce empty or garbled text. <br>• Password-protected PDFs cause extraction failure. |

---

#### Set Resume Data

| Property | Detail |
|----------|--------|
| **Type** | `n8n-nodes-base.set` (v3.4) |
| **Technical Role** | Bundles resume text, target role, and sender phone into a clean payload |
| **Assignments** | <ul><li>`resume_text` ← `{{ $json.text }}`</li><li>`target_role` ← `{{ $('Conversation State Manager').item.json.session.target_role }}`</li><li>`from` ← `{{ $('Conversation State Manager').item.json.from }}`</li></ul> |
| **Input** | Output of Extract Resume Text |
| **Output** | Item with `resume_text`, `target_role`, `from` |

---

### Block 4 — Scoring & AI Analysis

**Overview:**  
Two parallel branches process the resume: a deterministic scoring engine calculates a 0–100 career score with percentile ranking, while OpenAI GPT‑4o‑mini generates human-readable rejection reasons and actionable fixes.

**Nodes Involved:**  
- Scoring Engine  
- OpenAI – Rejection Analysis  
- Parse AI Response  

---

#### Scoring Engine

| Property | Detail |
|----------|--------|
| **Type** | `n8n-nodes-base.code` (v2) |
| **Technical Role** | Deterministic five-dimension scoring algorithm plus percentile calculation |
| **Scoring Dimensions** | <ul><li>**Impact (0–25)**: Unique impact verbs + quantitative metrics (%, $, users, etc.)</li><li>**Relevance (0–25)**: Keyword overlap between resume text and a role-specific keyword map (17+ roles). Falls back to generic keywords if no role match.</li><li>**ATS Readiness (0–20)**: Presence of common sections, email, phone, LinkedIn, GitHub, low special-character density.</li><li>**Clarity (0–15)**: Average sentence length, bullet point count, year-range mentions.</li><li>**Completeness (0–15)**: Word count band, education keywords, certification keywords, project mentions.</li></ul> |
| **Additional Logic** | <ul><li>Saves the final score to Supabase `resume_scores` table (upsert on `phone` + `target_role`).</li><li>Fetches all scores for the same target role to compute rank, total candidates, and percentile.</li><li>Derives interview probability (High / Moderate / Low / Very Low).</li><li>Extracts email and LinkedIn URL from resume text.</li><li>PATCHes `whatsapp_sessions` with extracted email, LinkedIn, and resume_url.</li></ul> |
| **Expressions / Variables** | `SUPABASE_URL`, `SUPABASE_KEY` (placeholders); references `$input.first().json.resume_text`, `.target_role`, `.from` |
| **Output Fields** | `finalScore`, `percentile`, `percentileLabel`, `rank`, `totalCandidates`, `probability`, `impactScore`, `relevanceScore`, `atsScore`, `clarityScore`, `completenessScore`, `bestRoleMatch`, `wordCount`, `extractedEmail`, `extractedLinkedin`, `targetRole`, `from`, `resume_text` |
| **Potential Failures** | • Supabase queries may fail (errors caught, logged). <br>• Unusual resume formats may produce low or zero scores. <br>• Role keyword map may not cover niche roles (fallback to generic keywords). |

---

#### OpenAI – Rejection Analysis

| Property | Detail |
|----------|--------|
| **Type** | `n8n-nodes-base.httpRequest` (v4.2) |
| **Technical Role** | Calls OpenAI Chat Completions API (GPT‑4o‑mini) to generate rejection reasons, fixes, and a one-liner summary |
| **Configuration** | <ul><li>`url`: `YOUR_API_ENDPOINT_HERE` (replace with `https://api.openai.com/v1/chat/completions` or a proxy)</li><li>`method`: POST</li><li>`authentication`: predefinedCredentialType → `openAiApi`</li><li>`sendBody`: true</li><li>`specifyBody`: json</li><li>`jsonBody`: Expression constructing request with `model: gpt-4o-mini`, `max_tokens: 800`, `response_format: {type: json_object}`, system message, and user message containing target role + truncated resume text (first 3000 chars)</li></ul> |
| **Prompt Structure** | System: “You are an expert ATS resume reviewer. Return ONLY valid JSON, no extra text.”<br>User: “Analyze this resume for the role of {target_role}. Resume: {resume_text}. Return ONLY this JSON: { rejection_reasons: [{reason}], fixes: [{fix}], one_liner: ‘one sentence summary’ }” |
| **Credentials** | OpenAI API key |
| **Output** | Full OpenAI response object (`choices[0].message.content`) |
| **Potential Failures** | • Invalid API key or quota exceeded. <br>• Model may not respect JSON-only instruction (Parse AI Response will fail). <br>• Resume text truncation may omit important details. <br>• Timeout on long requests. |

---

#### Parse AI Response

| Property | Detail |
|----------|--------|
| **Type** | `n8n-nodes-base.set` (v3.4) |
| **Technical Role** | Parses the JSON string from the OpenAI response into a structured object |
| **Assignments** | `ai_analysis` ← `{{ JSON.parse($json.choices[0].message.content) }}` |
| **Potential Failures** | • If OpenAI returns non-JSON, `JSON.parse` will throw an error and halt the branch. |

---

### Block 5 — Result Merging, Formatting & Delivery

**Overview:**  
Combines the deterministic scoring data with the AI-generated insights, formats a rich WhatsApp message, resets the user session to IDLE, and delivers the final report.

**Nodes Involved:**  
- Merge Score + AI  
- Format Result Message  
- Send Final Result  

---

#### Merge Score + AI

| Property | Detail |
|----------|--------|
| **Type** | `n8n-nodes-base.merge` (v3) |
| **Technical Role** | Combines the two parallel branches into a single item containing all fields |
| **Mode** | `combine` with `combineBy: combineAll` |
| **Inputs** | Input 0: Scoring Engine output; Input 1: Parse AI Response output |
| **Edge Cases** | • If either branch fails, the merge will wait indefinitely (no timeout by default). |

---

#### Format Result Message

| Property | Detail |
|----------|--------|
| **Type** | `n8n-nodes-base.code` (v2) |
| **Technical Role** | Builds the final WhatsApp message and resets the user session |
| **Message Structure** | <ul><li>Header: *CAREER RANK REPORT*</li><li>Role line</li><li>Score with 10-block visual bar (█/░)</li><li>Percentile with color emoji (🟢🟡🟠🔴)</li><li>Rank out of total candidates</li><li>Interview probability (High/Moderate/Low/Very Low)</li><li>Top 3 rejection reasons from AI</li><li>One-liner summary</li><li>Top 3 fixes from AI</li><li>Call-to-action: “Get personalized job recommendations 👇 https://meracareer.io”</li></ul> |
| **Session Reset** | PATCHes `whatsapp_sessions` setting `state: 'IDLE'` and `updated_at` to now |
| **Expressions / Variables** | `SUPABASE_URL`, `SUPABASE_KEY` (placeholders); references merged item fields |
| **Output** | `{ from, message }` |
| **Potential Failures** | • Session reset PATCH may fail (caught, logged). <br>• Message could exceed WhatsApp character limit (unlikely with 3 items each). |

---

#### Send Final Result

| Property | Detail |
|----------|--------|
| **Type** | `n8n-nodes-base.whatsApp` (v1) |
| **Technical Role** | Delivers the formatted career rank report to the user on WhatsApp |
| **Configuration** | <ul><li>`textBody`: `{{ $json.message }}`</li><li>`recipientPhoneNumber`: `{{ $json.from }}`</li><li>`phoneNumberId`: `YOUR_PHONE_NUMBER_HERE` (must be replaced)</li></ul> |
| **Credentials** | WhatsApp Business Cloud API |
| **Edge Cases** | Same WhatsApp limits as other send nodes. |

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|-----------|-----------|-----------------|----------------|----------------|-------------|
| Main Sticky | stickyNote | Visual documentation of overall workflow purpose | — | — | ## WhatsApp Career Ranking Bot — Analyze resumes via WhatsApp to provide career scores, ATS feedback, and ranking insights. How it works: 1. User triggers bot via keyword. 2. Collects name, target role, and PDF resume. 3. Scoring Engine calculates metrics. 4. AI identifies rejection reasons. 5. Returns personalized report to user. |
| WhatsApp Trigger | whatsAppTrigger | Entry point – receives incoming WhatsApp messages | — | Conversation State Manager | ## 1. Trigger & State — Handles WhatsApp webhook and manages user conversation progress in Supabase. |
| Conversation State Manager | code | State machine orchestrator; reads/writes Supabase session, determines next action and reply | WhatsApp Trigger | Route Action | ## 1. Trigger & State — Handles WhatsApp webhook and manages user conversation progress in Supabase. |
| Route Action | switch | Routes based on action: PROCESS_RESUME or other replies | Conversation State Manager | Send Processing Message, Send Conversation Reply | ## 1. Trigger & State — Handles WhatsApp webhook and manages user conversation progress in Supabase. |
| Send Conversation Reply | whatsApp | Sends text replies for non-processing conversation steps | Route Action (Send Reply) | — | ## 1. Trigger & State — Handles WhatsApp webhook and manages user conversation progress in Supabase. |
| Send Processing Message | whatsApp | Sends “Analyzing…” message when resume is received | Route Action (Process Resume) | Get Resume URL | ## 1. Trigger & State — Handles WhatsApp webhook and manages user conversation progress in Supabase. |
| Get Resume URL | whatsApp | Resolves WhatsApp media ID to a downloadable URL | Send Processing Message | Download Resume | ## 2. Processing & Analysis — Downloads resume, extracts text, performs scoring, and generates AI improvement suggestions. |
| Download Resume | httpRequest | Downloads the PDF binary from WhatsApp media server | Get Resume URL | Extract Resume Text, Upload Resume to Storage | ## 2. Processing & Analysis — Downloads resume, extracts text, performs scoring, and generates AI improvement suggestions. |
| Upload Resume to Storage | code | Uploads PDF to Supabase Storage and persists the public URL | Download Resume | — | ## 2. Processing & Analysis — Downloads resume, extracts text, performs scoring, and generates AI improvement suggestions. |
| Extract Resume Text | extractFromFile | Extracts plain text from the downloaded PDF | Download Resume | Set Resume Data | ## 2. Processing & Analysis — Downloads resume, extracts text, performs scoring, and generates AI improvement suggestions. |
| Set Resume Data | set | Bundles resume text, target role, and sender phone for downstream nodes | Extract Resume Text | Scoring Engine, OpenAI - Rejection Analysis | ## 2. Processing & Analysis — Downloads resume, extracts text, performs scoring, and generates AI improvement suggestions. |
| Scoring Engine | code | Calculates five-dimension career score and percentile; saves to Supabase | Set Resume Data | Merge Score + AI | ## 2. Processing & Analysis — Downloads resume, extracts text, performs scoring, and generates AI improvement suggestions. |
| OpenAI - Rejection Analysis | httpRequest | Calls OpenAI GPT‑4o‑mini for rejection reasons and fix suggestions | Set Resume Data | Parse AI Response | ## 2. Processing & Analysis — Downloads resume, extracts text, performs scoring, and generates AI improvement suggestions. |
| Parse AI Response | set | Parses OpenAI JSON response into structured object | OpenAI - Rejection Analysis | Merge Score + AI | ## 2. Processing & Analysis — Downloads resume, extracts text, performs scoring, and generates AI improvement suggestions. |
| Merge Score + AI | merge | Combines scoring engine output and AI analysis into one item | Scoring Engine, Parse AI Response | Format Result Message | ## 2. Processing & Analysis — Downloads resume, extracts text, performs scoring, and generates AI improvement suggestions. |
| Format Result Message | code | Formats final WhatsApp report and resets user session | Merge Score + AI | Send Final Result | ## 2. Processing & Analysis — Downloads resume, extracts text, performs scoring, and generates AI improvement suggestions. |
| Send Final Result | whatsApp | Delivers the formatted career rank report to the WhatsApp user | Format Result Message | — | ## 2. Processing & Analysis — Downloads resume, extracts text, performs scoring, and generates AI improvement suggestions. |
| Section 1 | stickyNote | Visual label for Trigger & State block | — | — | ## 1. Trigger & State — Handles WhatsApp webhook and manages user conversation progress in Supabase. |
| Section 2 | stickyNote | Visual label for Processing & Analysis block | — | — | ## 2. Processing & Analysis — Downloads resume, extracts text, performs scoring, and generates AI improvement suggestions. |

---

## 4. Reproducing the Workflow from Scratch

Follow these steps in order to recreate the entire workflow in a fresh n8n instance.

### Prerequisites

1. **WhatsApp Business Cloud API** – Create a Meta app, add the WhatsApp product, obtain a permanent token and Phone Number ID, and put the app in Live mode (requires Business Verification).
2. **Supabase project** – Create a project, note the Project URL and service_role or anon key. Create the required tables and storage bucket (see below).
3. **OpenAI API key** – Obtain an API key with access to `gpt-4o-mini`.
4. **n8n instance** – Self-hosted or n8n Cloud.

### Supabase Setup

1. **Table `whatsapp_sessions`**

| Column | Type | Default / Notes |
|--------|------|------------------|
| `phone` | text | Primary key |
| `state` | text | Default `'IDLE'` |
| `name` | text | |
| `target_role` | text | |
| `started_at` | timestamptz | |
| `updated_at` | timestamptz | |
| `email` | text | (added by Scoring Engine) |
| `linkedin_url` | text | (added by Scoring Engine) |
| `resume_url` | text | (added by Upload Resume to Storage) |

2. **Table `resume_scores`**

| Column | Type | Notes |
|--------|------|-------|
| `phone` | text | Part of unique constraint with `target_role` |
| `target_role` | text | |
| `score` | integer | |
| `scored_at` | timestamptz | |

3. **Storage bucket `resumes`** – Create a public bucket named `resumes` in Supabase Storage; ensure RLS or service-role access allows uploads.

### Node Creation & Configuration

#### Step 1 – WhatsApp Trigger
1. Add a **WhatsApp Trigger** node.
2. Set **Updates** to `messages`.
3. Connect the **WhatsApp Business Cloud API** credential (token + phone number ID).
4. Note the webhook URL that n8n generates; configure it in Meta Developer dashboard under WhatsApp → Webhooks (subscribe to `messages`).

#### Step 2 – Conversation State Manager
1. Add a **Code** node, rename it to `Conversation State Manager`.
2. Set **Mode** to `Run Once for All Items` (default).
3. Paste the full JavaScript code from the original workflow.
4. Replace the placeholders:
   - `SUPABASE_URL` → your Supabase project URL (e.g., `https://xyzcompany.supabase.co`)
   - `SUPABASE_KEY` → your Supabase service_role or anon key
5. Connect: **WhatsApp Trigger → Conversation State Manager**.

#### Step 3 – Route Action
1. Add a **Switch** node, rename to `Route Action`.
2. Add two output rules:
   - **Output “Process Resume”**: condition `action` equals `PROCESS_RESUME` (string, case-sensitive).
   - **Output “Send Reply”**: two conditions combined with AND:
     - `action` does NOT equal `PROCESS_RESUME`
     - `action` does NOT equal `IGNORE`
3. Set **Fallback Output** to `none`.
4. Connect: **Conversation State Manager → Route Action**.

#### Step 4 – Send Conversation Reply
1. Add a **WhatsApp** node, rename to `Send Conversation Reply`.
2. Set **Operation** to `send`.
3. Set **Text Body** to `{{ $json.replyText }}`.
4. Set **Recipient Phone Number** to `{{ $json.from }}`.
5. Set **Phone Number ID** to your WhatsApp Phone Number ID (replace `YOUR_PHONE_NUMBER_HERE`).
6. Connect **WhatsApp Business Cloud API** credential.
7. Connect: **Route Action (Send Reply) → Send Conversation Reply**.

#### Step 5 – Send Processing Message
1. Add a **WhatsApp** node, rename to `Send Processing Message`.
2. Same configuration as Step 4 (uses `replyText` and `from` from the PROCESS_RESUME action).
3. Connect: **Route Action (Process Resume) → Send Processing Message**.

#### Step 6 – Get Resume URL
1. Add a **WhatsApp** node, rename to `Get Resume URL`.
2. Set **Resource** to `media`.
3. Set **Operation** to `mediaUrlGet`.
4. Set **Media Get ID** to `{{ $('Conversation State Manager').item.json.documentId }}`.
5. Connect **WhatsApp Business Cloud API** credential.
6. Connect: **Send Processing Message → Get Resume URL**.

#### Step 7 – Download Resume
1. Add an **HTTP Request** node, rename to `Download Resume`.
2. Set **URL** to `{{ $json.url }}`.
3. Under **Options → Response**, set **Response Format** to `file`.
4. Set **Authentication** to `Predefined Credential Type` → `whatsAppApi`.
5. Connect: **Get Resume URL → Download Resume**.

#### Step 8 – Upload Resume to Storage
1. Add a **Code** node, rename to `Upload Resume to Storage`.
2. Paste the full JavaScript code from the original workflow.
3. Replace `SUPABASE_URL` and `SUPABASE_KEY` placeholders with your values.
4. Connect: **Download Resume → Upload Resume to Storage** (second output branch).

#### Step 9 – Extract Resume Text
1. Add an **Extract From File** node, rename to `Extract Resume Text`.
2. Set **Operation** to `pdf`.
3. Leave all other options as default.
4. Connect: **Download Resume → Extract Resume Text** (first output branch).

#### Step 10 – Set Resume Data
1. Add a **Set** node, rename to `Set Resume Data`.
2. Add three assignments:
   - `resume_text` (string) = `{{ $json.text }}`
   - `target_role` (string) = `{{ $('Conversation State Manager').item.json.session.target_role }}`
   - `from` (string) = `{{ $('Conversation State Manager').item.json.from }}`
3. Connect: **Extract Resume Text → Set Resume Data**.

#### Step 11 – Scoring Engine
1. Add a **Code** node, rename to `Scoring Engine`.
2. Paste the full JavaScript code from the original workflow.
3. Replace `SUPABASE_URL` and `SUPABASE_KEY` placeholders with your values.
4. Connect: **Set Resume Data → Scoring Engine**.

#### Step 12 – OpenAI – Rejection Analysis
1. Add an **HTTP Request** node, rename to `OpenAI - Rejection Analysis`.
2. Set **Method** to `POST`.
3. Set **URL** to `https://api.openai.com/v1/chat/completions` (or your proxy endpoint).
4. Set **Authentication** to `Predefined Credential Type` → `openAiApi`; connect your OpenAI credential.
5. Enable **Send Body** and set **Specify Body** to `JSON`.
6. Set **JSON Body** to the expression from the original workflow (constructs model `gpt-4o-mini`, max_tokens 800, JSON mode, system prompt, and user prompt with target role + resume text).
7. Connect: **Set Resume Data → OpenAI - Rejection Analysis**.

#### Step 13 – Parse AI Response
1. Add a **Set** node, rename to `Parse AI Response`.
2. Add one assignment:
   - `ai_analysis` (object) = `{{ JSON.parse($json.choices[0].message.content) }}`
3. Connect: **OpenAI - Rejection Analysis → Parse AI Response**.

#### Step 14 – Merge Score + AI
1. Add a **Merge** node, rename to `Merge Score + AI`.
2. Set **Mode** to `Combine`.
3. Set **Combine By** to `Combine All`.
4. Connect:
   - **Scoring Engine → Merge Score + AI** (input 0)
   - **Parse AI Response → Merge Score + AI** (input 1)

#### Step 15 – Format Result Message
1. Add a **Code** node, rename to `Format Result Message`.
2. Paste the full JavaScript code from the original workflow.
3. Replace `SUPABASE_URL` and `SUPABASE_KEY` placeholders with your values.
4. Connect: **Merge Score + AI → Format Result Message**.

#### Step 16 – Send Final Result
1. Add a **WhatsApp** node, rename to `Send Final Result`.
2. Set **Operation** to `send`.
3. Set **Text Body** to `{{ $json.message }}`.
4. Set **Recipient Phone Number** to `{{ $json.from }}`.
5. Set **Phone Number ID** to your WhatsApp Phone Number ID (replace `YOUR_PHONE_NUMBER_HERE`).
6. Connect **WhatsApp Business Cloud API** credential.
7. Connect: **Format Result Message → Send Final Result**.

### Final Connection Verification

Verify the complete connection chain matches:

```
WhatsApp Trigger → Conversation State Manager → Route Action
Route Action (Process Resume) → Send Processing Message → Get Resume URL → Download Resume
Download Resume → Extract Resume Text → Set Resume Data
Download Resume → Upload Resume to Storage
Set Resume Data → Scoring Engine → Merge Score + AI (input 0)
Set Resume Data → OpenAI - Rejection Analysis → Parse AI Response → Merge Score + AI (input 1)
Merge Score + AI → Format Result Message → Send Final Result
Route Action (Send Reply) → Send Conversation Reply
```

### Credential Configuration Summary

| Credential | Service | Required Fields |
|------------|---------|-----------------|
| WhatsApp Business Cloud API | Meta | Access Token, Phone Number ID |
| WhatsApp API (for HTTP node) | Meta | Same as above (used by Download Resume) |
| OpenAI API | OpenAI | API Key |
| Supabase (used via REST in Code nodes) | Supabase | Project URL, Service Role Key (or anon key if RLS allows) |

### Important Placeholders to Replace

| Placeholder | Location | Replace With |
|-------------|----------|---------------|
| `YOUR_PHONE_NUMBER_HERE` | Send Conversation Reply, Send Processing Message, Send Final Result | Your WhatsApp Business Phone Number ID |
| `YOUR_API_ENDPOINT_HERE` | OpenAI - Rejection Analysis | `https://api.openai.com/v1/chat/completions` (or custom proxy) |
| `eyJ_YOUR_JWT_TOKEN_HERE` | Conversation State Manager, Scoring Engine, Format Result Message, Upload Resume to Storage | Your Supabase service_role or anon JWT |
| `https://zfyyiewkkjyzxnyiabvk.supabase.co` | All Code nodes referencing Supabase | Your own Supabase project URL |

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|--------------|-----------------|
| The bot only supports PDF resumes; DOC/DOCX uploads are rejected with a helpful message instructing the user to attach a PDF. | Conversation State Manager state `WAITING_RESUME` |
| WhatsApp Business API must be in **Live mode** (not Sandbox) to receive messages from non-whitelisted numbers — requires completed Meta Business Verification. | Meta Developer documentation |
| Session state persists in Supabase, allowing users to resume mid-flow if they disconnect. Each phone number is a unique session key. | `whatsapp_sessions` table |
| The scoring engine uses a fixed keyword map for 17+ job roles; for roles not in the map, it falls back to generic keywords. This can be extended by editing the `roleKeywordsMap` object inside the Scoring Engine code node. | Scoring Engine |
| OpenAI is called with `response_format: { type: "json_object" }` and `max_tokens: 800`. The prompt explicitly requests JSON only; if the model deviates, the Parse AI Response node will fail. | OpenAI - Rejection Analysis |
| Resume text sent to OpenAI is truncated to the first 3000 characters to stay within token limits and avoid excessive cost. | OpenAI - Rejection Analysis jsonBody expression |
| The final message includes a call-to-action link: https://meracareer.io — update or remove this link in the Format Result Message code node to match your own product. | Format Result Message |
| Supabase Storage bucket `resumes` must be created and set to public (or accessible via service_role key) for the Upload Resume to Storage node to work correctly. | Supabase Storage setup |
| The workflow handles concurrent users independently because each session is keyed by the sender’s phone number. No locking mechanism is implemented; race conditions are unlikely given per-user state but could occur if a user sends multiple rapid messages. | Conversation State Manager |
| To add analytics, insert a Supabase INSERT node after the Scoring Engine to log each submission with timestamp, score, and role. The `resume_scores` table already captures score and role. | Customization suggestion |
| To offer a paid detailed report or booking link after the free score, extend the conversation state machine (add a new state) and add additional WhatsApp Send nodes after Send Final Result. | Customization suggestion |
| The Scoring Engine also extracts the candidate’s email and LinkedIn URL from the resume and saves them to the session record — useful for downstream CRM integration. | Scoring Engine |
| If the WhatsApp media URL expires before the Download Resume node executes, the download will fail. Consider adding a retry or error-handling branch in production. | Download Resume potential issue |
| The Format Result Message node resets the user session to `IDLE` after sending the report, so the user can start a new evaluation by sending `CHECK MY RANK` again. | Format Result Message |
| Meta’s WhatsApp messaging rate limits apply; sending too many messages in a short window can result in throttling or temporary blocks. | WhatsApp Business API limits |