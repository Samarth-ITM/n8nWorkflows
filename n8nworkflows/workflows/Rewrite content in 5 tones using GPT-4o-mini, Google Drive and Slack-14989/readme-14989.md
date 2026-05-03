Rewrite content in 5 tones using GPT-4o-mini, Google Drive and Slack

https://n8nworkflows.xyz/workflows/rewrite-content-in-5-tones-using-gpt-4o-mini--google-drive-and-slack-14989


# Rewrite content in 5 tones using GPT-4o-mini, Google Drive and Slack

The user wants me to analyze an n8n workflow JSON and produce a comprehensive structured reference document in English, following a specific structure with sections:

1. Workflow Overview
2. Block-by-Block Analysis
3. Summary Table
4. Reproducing the Workflow from Scratch
5. General Notes & Resources

I need to go through each node, understand its function, connections, config, expressions, and provide detailed analysis. Also need to handle sticky notes and include them in the summary table's "Sticky Note" column.

Let me parse the workflow JSON and identify all nodes, connections, sticky notes.

Nodes:
1. "Overview" - stickyNote, color 4, content about AI Content Tone Rewriter
2. "Section — Form Input and Config" - stickyNote, color 5
3. "Section — AI Tone Rewriting" - stickyNote, color 6
4. "Section — Document Assembly" - stickyNote, color 6
5. "Section — Google Drive Upload" - stickyNote, color 4
6. "Section — Slack Preview" - stickyNote, color 4
7. "Note — Edit Config Before Activating" - stickyNote, color 3
8. "1. Form — Content Tone Rewriter" - formTrigger, v2.2
9. "2. Set — Config Values" - set, v3.4
10. "3. AI Agent — Tone Rewriter" - @n8n/n8n-nodes-langchain.agent, v1.7
11. "4. OpenAI — GPT-4o-mini Model" - @n8n/n8n-nodes-langchain.lmChatOpenAi, v1.2
12. "5. Parser — Structured Tone Output" - @n8n/n8n-nodes-langchain.outputParserStructured, v1.3
13. "6. Code — Assemble Document Content" - n8n-nodes-base.code, v2
14. "7. Code — Convert to Binary" - n8n-nodes-base.code, v2
15. "8. Google Drive — Upload Tone Doc" - n8n-nodes-base.googleDrive, v3
16. "9. Slack — Send Concise Preview" - n8n-nodes-base.slack, v2.2

Connections:
- "1. Form — Content Tone Rewriter" -> "2. Set — Config Values" (main)
- "2. Set — Config Values" -> "3. AI Agent — Tone Rewriter" (main)
- "4. OpenAI — GPT-4o-mini Model" -> "3. AI Agent — Tone Rewriter" (ai_languageModel)
- "5. Parser — Structured Tone Output" -> "3. AI Agent — Tone Rewriter" (ai_outputParser)
- "3. AI Agent — Tone Rewriter" -> "6. Code — Assemble Document Content" (main)
- "6. Code — Assemble Document Content" -> "7. Code — Convert to Binary" (main, index 0)
- "6. Code — Assemble Document Content" -> "9. Slack — Send Concise Preview" (main, index 1)
- "7. Code — Convert to Binary" -> "8. Google Drive — Upload Tone Doc" (main)

So the flow is:
Form -> Set Config -> AI Agent (with OpenAI model + Structured Output Parser) -> Code Assemble -> splits to (Code Convert to Binary -> Google Drive Upload) and (Slack Send Concise Preview)

The AI Agent has a language model connection from OpenAI node and output parser connection from Structured Output Parser node.

Now I need to map sticky notes to nodes. The sticky notes are labeled with section titles covering multiple nodes. Let me examine positions:

- "Overview" at [-768,1152] - large overview note
- "Section — Form Input and Config" at [-160,1488] - covers nodes 1 (Form) and 2 (Set Config)
- "Section — AI Tone Rewriting" at [384,1360] - covers nodes 3, 4, 5 (AI Agent, OpenAI Model, Parser)
- "Section — Document Assembly" at [944,1488] - covers node 6 (Code Assemble)
- "Section — Google Drive Upload" at [1456,912] - covers nodes 7 and 8 (Code Convert to Binary and Google Drive Upload)
- "Section — Slack Preview" at [1568,1680] - covers node 9 (Slack)
- "Note — Edit Config Before Activating" at [32,2128] - warning about editing config values in node 2

So for summary table, sticky note content should be duplicated for all nodes in that section.

Now I need to produce the document. Let me structure it carefully.

### 1. Workflow Overview

The workflow is a content tone rewriter. It receives input from a form (original text, content type, project name, target audience), configures parameters, uses GPT-4o-mini via an AI Agent with a structured output parser to generate five tone variants (professional, casual, persuasive, concise, empathetic), assembles them into a document, converts to binary, uploads to Google Drive, and sends a Slack preview of the concise version.

Logical blocks:
- Block 1: Input Reception (Form trigger)
- Block 2: Configuration (Set node)
- Block 3: AI Processing (AI Agent with OpenAI model and structured output parser)
- Block 4: Document Assembly (Code node)
- Block 5: Output Delivery (Convert to Binary, Google Drive upload, Slack notification)

Actually, based on the sticky notes, I might group them similarly:
1.1 Form Input and Config
1.2 AI Tone Rewriting
1.3 Document Assembly
1.4 Google Drive Upload
1.5 Slack Preview

Now I need detailed node-by-node analysis.

Let me detail each node:

**1. Form — Content Tone Rewriter** (formTrigger, v2.2)
- Webhook ID: 0e7d0337-964d-43a9-9df5-f2bc65659d98
- Form title: "AI Content Tone Rewriter"
- Form description: "Paste your text below. AI will rewrite it in 5 different tones and save all versions to Google Drive."
- Form fields:
  1. "Your Original Text" - textarea, required, placeholder for pasting text
  2. "Content Type" - dropdown, required, options: Email, Social Media Caption, Blog Paragraph, Client Proposal, Product Description, Other
  3. "Your Name or Project Name" - required, placeholder: "e.g. Rahul — Incrementors Proposal"
  4. "Target Audience" - optional, placeholder: "e.g. C-suite executives, young professionals, general public"
- Output: JSON with those four field names.
- Connects to: "2. Set — Config Values"

**2. Set — Config Values** (set, v3.4)
- Assignments:
  - folderId: "YOUR_GOOGLE_DRIVE_FOLDER_ID" (string placeholder)
  - slackChannel: "#YOUR-SLACK-CHANNEL" (string placeholder)
  - companyName: "YOUR COMPANY NAME" (string placeholder)
  - originalText: expression `={{ $json['Your Original Text'] }}`
  - contentType: expression `={{ $json['Content Type'] }}`
  - projectName: expression `={{ $json['Your Name or Project Name'] }}`
  - targetAudience: expression `={{ $json['Target Audience'] || 'General audience' }}` (defaults if empty)
  - docTitle: expression `=Tone Rewrites — {{ $json['Your Name or Project Name'] }} — {{ $now.toFormat('dd MMM yyyy') }}`
  - runDate: expression `={{ $now.toFormat('dd MMM yyyy HH:mm') }}`
- Connects to: "3. AI Agent — Tone Rewriter"

**3. AI Agent — Tone Rewriter** (@n8n/n8n-nodes-langchain.agent, v1.7)
- Text/prompt: A detailed system prompt instructing GPT-4o-mini to rewrite the text in five tones and return a JSON object with keys: professional, casual, persuasive, concise, empathetic. Uses expressions for companyName, originalText, contentType, targetAudience.
- promptType: "define" (so text is the system prompt)
- hasOutputParser: true (connected to structured output parser)
- Connections:
  - Input from: "2. Set — Config Values" (main)
  - ai_languageModel from: "4. OpenAI — GPT-4o-mini Model"
  - ai_outputParser from: "5. Parser — Structured Tone Output"
  - Output to: "6. Code — Assemble Document Content"

**4. OpenAI — GPT-4o-mini Model** (@n8n/n8n-nodes-langchain.lmChatOpenAi, v1.2)
- Model: gpt-4o-mini (selected from list)
- Options: maxTokens: 1500, temperature: 0.7
- Credential: requires OpenAI API key credential (not specified in JSON, must be configured)
- Connects to: "3. AI Agent — Tone Rewriter" as ai_languageModel

**5. Parser — Structured Tone Output** (@n8n/n8n-nodes-langchain.outputParserStructured, v1.3)
- schemaType: "manual"
- inputSchema: JSON schema object with required properties: professional (string), casual (string), persuasive (string), concise (string), empathetic (string)
- Connects to: "3. AI Agent — Tone Rewriter" as ai_outputParser

**6. Code — Assemble Document Content** (n8n-nodes-base.code, v2)
- JS code:
  - Reads aiOutput from $input.first().json.output
  - Reads config from $('2. Set — Config Values').item.json
  - Uses fallback strings if any tone version is missing
  - Builds docContent string with header, original text, five tone versions separated by lines
  - Builds slackPreview string showing project name, content type, date, and concise version
  - Returns JSON with docContent, docTitle, folderId, slackChannel, slackPreview, and individual tone versions, projectName
- Connects to: "7. Code — Convert to Binary" (main 0) and "9. Slack — Send Concise Preview" (main 1)

**7. Code — Convert to Binary** (n8n-nodes-base.code, v2)
- JS code:
  - Reads item from input
  - Converts docContent to base64 using Buffer.from(textContent, 'utf-8').toString('base64')
  - Returns JSON with docTitle, folderId, slackChannel, slackPreview, projectName plus binary data with mimeType text/plain, fileName docTitle + '.txt', extension txt
- Connects to: "8. Google Drive — Upload Tone Doc"

**8. Google Drive — Upload Tone Doc** (n8n-nodes-base.googleDrive, v3)
- Parameters:
  - name: `={{ $json.docTitle }}`
  - driveId: mode list, value "My Drive"
  - folderId: mode id, value `={{ $json.folderId }}`
  - options: {}
- Credential: Google Drive OAuth2 (must be configured)
- Input binary: expects binary data from node 7
- No explicit output connections shown (terminal node for that branch)

**9. Slack — Send Concise Preview** (n8n-nodes-base.slack, v2.2)
- Parameters:
  - text: `={{ $json.slackPreview }}`
  - otherOptions.mrkdwn: true (markdown formatting enabled)
  - authentication: "oAuth2"
- Channel: determined by slackChannel field in data? Actually text uses $json.slackPreview which contains the channel info indirectly? Wait, the node uses a channel parameter? Let me check - The node parameters only show text and otherOptions. The channel may be set via the channel parameter, but it's not visible. Actually looking at the JSON, the slack node parameters don't show a "channel" field explicitly. In n8n slack node v2.2, there's usually a "channel" parameter. It might be using the data field slackChannel? Actually, looking at the code node that builds slackPreview, it includes project info but not channel. The slackChannel is passed in the JSON but the node may use it via expression. But the node parameters only show `text` and `otherOptions`. There's no explicit channel parameter visible. Possibly the channel is set via the "channel" parameter that's not shown because it uses the data field? Let me check - In Slack node v2.2, there is usually a channel parameter that can be an expression. In this JSON, it's not shown; maybe it's set to use `={{ $json.slackChannel }}` but it's not visible. Since the node's parameters show only `text` and `otherOptions`, I need to note that the channel must be configured, likely using the slackChannel config variable. I'll mention it needs to be set.

Actually, wait - The parameters may have other fields not shown. In the JSON, `parameters` contains `text` and `otherOptions`. The `channel` field might be missing because n8n's default might use the default channel or maybe they've configured it elsewhere. I'll note that the channel should be set to the slackChannel value.

- Credential: Slack OAuth2 (must be configured)
- Connects to: terminal node (no further connections)

Now for sticky notes mapping:

- "Overview" sticky note at [-768,1152] - overview description covering entire workflow. This might be displayed near the top and could be associated with all nodes? But typical mapping: it's a general overview note, not specifically covering certain nodes. I could include it as a note for all nodes but that seems redundant. Instead, I'll mention it in the overview section but in the summary table I'll include it for no specific node (leave blank) since it's a general overview, not directly covering a specific node. Actually the instruction says: "If a sticky note visually covers multiple nodes, duplicate its content in each affected row." The Overview sticky note is large and may visually cover all nodes. It's at position [-768,1152] with width 540 and height 928. The nodes start at positions around x = -96 to 1664, y = 1040 to 2128. The overview note is to the left of all nodes, so it might not directly overlap visually. However, it's labeled "Overview" and may be placed next to the workflow. I'll consider it as a general overview note, not covering specific nodes.

- "Section — Form Input and Config" sticky note at [-160,1488] with size 484x372 - covers nodes 1 (Form at [-96,1632]) and 2 (Set at [176,1632]). Both are within the x range -160 to 324 and y range 1488 to 1860. Node 1 is at -96,1632 which is inside. Node 2 is at 176,1632 which is inside. So I'll add this sticky content to both nodes.

- "Section — AI Tone Rewriting" sticky note at [384,1360] with size 404x708 - covers nodes 3 (480,1632), 4 (448,1840), 5 (624,1840). All within x 384-788, y 1360-2068. Yes, all three are inside.

- "Section — Document Assembly" sticky note at [944,1488] with size 512x340 - covers node 6 (1200,1632). 1200 is within 944-1456? Actually 944+512=1456. Node 6 at 1200 is within. So this note covers node 6.

- "Section — Google Drive Upload" sticky note at [1456,912] with size 544x516 - covers nodes 7 (1664,1040) and 8 (1664,1232). 1456+544=2000. Both nodes at x 1664 within range. Y 912+516=1428. Node 7 at y 1040 and node 8 at y 1232 are within. So this note covers nodes 7 and 8.

- "Section — Slack Preview" sticky note at [1568,1680] with size 496x452 - covers node 9 (1648,1872). 1568+496=2064. Node at 1648 within. Y 1680+452=2132. Node at 1872 within. So this note covers node 9.

- "Note — Edit Config Before Activating" sticky note at [32,2128] with size 1100x140 - might be placed near node 2? Node 2 is at [176,1632]. This note is lower down (y 2128) and may be near node 2 but not overlapping. I'll check if it's relevant. It says to edit three values in node 2. So I'll associate it with node 2.

Now I'll produce the document.

Let me structure the sections:

### 1. Workflow Overview

Explain purpose: multi-tone content rewriting, form input, AI processing, document assembly, Google Drive upload, Slack preview.

Logical blocks:
1.1 Input Reception and Configuration (Form + Set)
1.2 AI Processing (AI Agent + OpenAI Model + Structured Output Parser)
1.3 Document Assembly (Code node)
1.4 Binary Conversion and Google Drive Upload (Code + Google Drive)
1.5 Slack Notification (Slack)

### 2. Block-by-Block Analysis

For each block, describe overview, nodes involved, and node details.

### 3. Summary Table

List all nodes (including sticky notes as separate rows? No, sticky notes are not nodes per se but the instruction says include sticky note content in each affected row. I'll include sticky note content in the "Sticky Note" column for each node that is covered by a sticky note.

### 4. Reproducing the Workflow from Scratch

Step-by-step instructions to recreate.

### 5. General Notes & Resources

Notes about branding, support contact, etc.

Now let's write the document.

For node details, I'll describe:

- Type and technical role
- Configuration choices
- Key expressions or variables
- Input and output connections
- Version-specific requirements
- Edge cases or potential failure types
- Sub-workflow reference (none in this case)

Let me also note that the AI Agent uses structured output parser which ensures all five keys are present. If the AI fails to produce all five, the parser will throw an error.

The code nodes use Buffer which is available in n8n's Node.js environment.

The Google Drive node uses OAuth2 credential.

The Slack node uses OAuth2.

Edge cases:
- Form submission without target audience → defaults to "General audience"
- Missing AI response keys → fallback strings in code node
- OpenAI API errors, rate limits, token limits
- Google Drive folder ID incorrect → upload failure
- Slack channel name format issues
- Binary conversion errors

Now let's produce the final document. 1. Workflow Overview

**Purpose**  
This workflow lets a content team submit any piece of text through a browser form and instantly receive five professionally‑written tone variants — Professional, Casual, Persuasive, Concise, and Empathetic — generated by OpenAI’s GPT‑4o‑mini model. All five versions are assembled into a single document, uploaded to a designated Google Drive folder, and a short Slack preview of the Concise version is posted to a team channel. The workflow is designed for copywriters, agencies, and anyone who must repurpose the same core message for different audiences.

**Logical Blocks**

| Block | Role | Key Nodes |
|-------|------|-----------|
| 1.1 Input Reception & Configuration | Capture user input and prepare runtime variables | 1. Form — Content Tone Rewriter, 2. Set — Config Values |
| 1.2 AI Tone Rewriting | Call GPT‑4o‑mini with a strict five‑tone prompt and enforce structured output | 3. AI Agent — Tone Rewriter, 4. OpenAI — GPT‑4o‑mini Model, 5. Parser — Structured Tone Output |
| 1.3 Document Assembly | Merge AI output and metadata into a printable document and a Slack preview message | 6. Code — Assemble Document Content |
| 1.4 Binary Conversion & Google Drive Upload | Encode the assembled text as a binary file and upload it to Drive | 7. Code — Convert to Binary, 8. Google Drive — Upload Tone Doc |
| 1.5 Slack Notification | Post the Concise version preview to a Slack channel | 9. Slack — Send Concise Preview |

The flow runs sequentially from 1.1 through 1.3, then fans out in parallel to 1.4 and 1.5.

---

## 2. Block-by-Block Analysis

### Block 1.1 — Input Reception & Configuration

**Overview**  
The form trigger collects the source text, content type, project name, and an optional audience. The Set node normalises these fields, provides default values, and builds derived variables (document title, run timestamp) needed by downstream nodes.

**Nodes Involved**  
- 1. Form — Content Tone Rewriter  
- 2. Set — Config Values  

#### 1. Form — Content Tone Rewriter

| Attribute | Detail |
|-----------|--------|
| **Type** | `n8n-nodes-base.formTrigger` (v2.2) |
| **Technical Role** | Webhook‑based form that starts the workflow on submission |
| **Form Title** | “AI Content Tone Rewriter” |
| **Form Description** | “Paste your text below. AI will rewrite it in 5 different tones and save all versions to Google Drive.” |
| **Fields** | 1. **Your Original Text** – textarea, required, placeholder “Paste your email, blog paragraph, social caption, proposal, or any text here…”  <br>2. **Content Type** – dropdown, required, options: Email, Social Media Caption, Blog Paragraph, Client Proposal, Product Description, Other <br>3. **Your Name or Project Name** – text, required, placeholder “e.g. Rahul — Incrementors Proposal” <br>4. **Target Audience** – text, optional, placeholder “e.g. C-suite executives, young professionals, general public” |
| **Output** | JSON object with the four field names as keys |
| **Connections** | → 2. Set — Config Values (main) |
| **Edge Cases** | - If the form URL is accessed while the workflow is deactivated, n8n returns a 404. <br>- Empty “Target Audience” field is allowed; downstream node defaults it. |
| **Sticky Note** | “Form Input and Config — Form collects the original text, content type, project name, and target audience. Config stores Drive folder ID, Slack channel, company name, and builds the document title and run timestamp.” |

#### 2. Set — Config Values

| Attribute | Detail |
|-----------|--------|
| **Type** | `n8n-nodes-base.set` (v3.4) |
| **Technical Role** | Maps raw form fields into clean, reusable variables and stores user‑specific configuration |
| **Assignments** | • `folderId` – string literal “YOUR_GOOGLE_DRIVE_FOLDER_ID” (must be replaced) <br>• `slackChannel` – string literal “#YOUR-SLACK-CHANNEL” (must be replaced) <br>• `companyName` – string literal “YOUR COMPANY NAME” (must be replaced) <br>• `originalText` – expression `={{ $json['Your Original Text'] }}` <br>• `contentType` – expression `={{ $json['Content Type'] }}` <br>• `projectName` – expression `={{ $json['Your Name or Project Name'] }}` <br>• `targetAudience` – expression `={{ $json['Target Audience'] || 'General audience' }}` (defaults to “General audience”) <br>• `docTitle` – expression `=Tone Rewrites — {{ $json['Your Name or Project Name'] }} — {{ $now.toFormat('dd MMM yyyy') }}` <br>• `runDate` – expression `={{ $now.toFormat('dd MMM yyyy HH:mm') }}` |
| **Connections** | ← 1. Form — Content Tone Rewriter (main) <br>→ 3. AI Agent — Tone Rewriter (main) |
| **Edge Cases** | - If any of the three placeholder strings are not replaced, downstream Google Drive and Slack calls will fail (invalid folder ID or channel). <br>- Empty “Target Audience” is gracefully defaulted. |
| **Sticky Notes** | • “Form Input and Config — Form collects the original text, content type, project name, and target audience. Config stores Drive folder ID, Slack channel, company name, and builds the document title and run timestamp.” <br>• “⚠️ Edit This Node Before Activating — Replace three values: YOUR_GOOGLE_DRIVE_FOLDER_ID (from Drive folder URL after /folders/), YOUR-SLACK-CHANNEL (include the # symbol), and YOUR COMPANY NAME.” |

---

### Block 1.2 — AI Tone Rewriting

**Overview**  
An AI Agent calls GPT‑4o‑mini with a detailed system prompt that requests exactly five tone variants in a JSON object. The Structured Output Parser guarantees all five keys are present; if any are missing the workflow errors out.

**Nodes Involved**  
- 3. AI Agent — Tone Rewriter  
- 4. OpenAI — GPT-4o-mini Model  
- 5. Parser — Structured Tone Output  

#### 3. AI Agent — Tone Rewriter

| Attribute | Detail |
|-----------|--------|
| **Type** | `@n8n/n8n-nodes-langchain.agent` (v1.7) |
| **Technical Role** | Orchestrates the LLM call; receives config data, builds the prompt, and routes the response through the output parser |
| **Prompt** | System‑level text defined in the node (promptType = “define”). It instructs the model to act as a tone specialist for `{{ $json.companyName }}`, rewrite the original text (`{{ $json.originalText }}`) of type `{{ $json.contentType }}` for audience `{{ $json.targetAudience }}` into five tones, and return **only** a JSON object with keys `professional`, `casual`, `persuasive`, `concise`, `empathetic`. No markdown, no extra text. |
| **hasOutputParser** | true (connects to Parser — Structured Tone Output) |
| **Connections** | ← 2. Set — Config Values (main) <br>← 4. OpenAI — GPT-4o-mini Model (ai_languageModel) <br>← 5. Parser — Structured Tone Output (ai_outputParser) <br>→ 6. Code — Assemble Document Content (main) |
| **Edge Cases** | - If the model returns additional text outside the JSON object, the parser will reject it. <br>- Excessive token usage (>1500 max tokens) may truncate the response. <br>- API key errors, rate limits, or network issues will cause this node to fail. |
| **Sticky Note** | “AI Tone Rewriting — GPT-4o-mini rewrites the original text in five tones simultaneously. Structured Output Parser enforces that all five versions are always returned: Professional, Casual, Persuasive, Concise, and Empathetic.” |

#### 4. OpenAI — GPT-4o-mini Model

| Attribute | Detail |
|-----------|--------|
| **Type** | `@n8n/n8n-nodes-langchain.lmChatOpenAi` (v1.2) |
| **Technical Role** | Provides the LLM backend for the AI Agent |
| **Model** | `gpt-4o-mini` (selected from list) |
| **Options** | maxTokens: 1500, temperature: 0.7 |
| **Credential** | OpenAI API key (must be created and selected) |
| **Connections** | → 3. AI Agent — Tone Rewriter (ai_languageModel) |
| **Edge Cases** | - Insufficient API credit returns a 402 error. <br>- Temperature > 0.9 can increase creative variance but may reduce structured output reliability. |
| **Sticky Note** | Same as AI Agent block note. |

#### 5. Parser — Structured Tone Output

| Attribute | Detail |
|-----------|--------|
| **Type** | `@n8n/n8n-nodes-langchain.outputParserStructured` (v1.3) |
| **Technical Role** | Enforces a JSON schema on the model’s response; fails the workflow if any required key is missing |
| **Schema Type** | manual |
| **Input Schema** | `{ "type": "object", "properties": { "professional": {"type":"string","description":"Professional tone rewrite — formal and polished"}, "casual": {"type":"string","description":"Casual tone rewrite — warm and conversational"}, "persuasive": {"type":"string","description":"Persuasive tone rewrite — compelling and action-oriented"}, "concise": {"type":"string","description":"Concise tone rewrite — stripped to minimum words"}, "empathetic": {"type":"string","description":"Empathetic tone rewrite — warm and understanding"} }, "required": ["professional","casual","persuasive","concise","empathetic"] }` |
| **Connections** | → 3. AI Agent — Tone Rewriter (ai_outputParser) |
| **Edge Cases** | - If the LLM omits any key, the parser throws an error visible in the execution log. <br>- Extra keys are ignored but not harmful. |
| **Sticky Note** | Same as AI Agent block note. |

---

### Block 1.3 — Document Assembly

**Overview**  
A Code node consumes the AI output and config data, builds a complete plain‑text document with headers, dividers, and all five tone versions, and also prepares a Slack preview string containing the Concise version.

**Nodes Involved**  
- 6. Code — Assemble Document Content  

#### 6. Code — Assemble Document Content

| Attribute | Detail |
|-----------|--------|
| **Type** | `n8n-nodes-base.code` (v2) |
| **Technical Role** | Pure JavaScript transformation that creates the final document text and Slack message |
| **Key Logic** | 1. Reads `output` from the AI Agent (`$input.first().json.output`). <br>2. Pulls config data from node “2. Set — Config Values”. <br>3. Provides fallback strings for any missing tone version. <br>4. Constructs `docContent` with project metadata, original text, and five labelled tone sections separated by `=` dividers. <br>5. Constructs `slackPreview` showing project name, content type, date, and the Concise version. <br>6. Returns an object with `docContent`, `docTitle`, `folderId`, `slackChannel`, `slackPreview`, plus each individual tone for downstream use. |
| **Output Fields** | `docContent`, `docTitle`, `folderId`, `slackChannel`, `slackPreview`, `professional`, `casual`, `persuasive`, `concise`, `empathetic`, `projectName` |
| **Connections** | ← 3. AI Agent — Tone Rewriter (main) <br>→ 7. Code — Convert to Binary (main 0) <br>→ 9. Slack — Send Concise Preview (main 1) |
| **Edge Cases** | - If the AI returns `null` or an empty `output` object, the fallback strings “Could not generate … version.” are used. <br>- Very long original text may produce a document that exceeds Google Drive upload limits (rare for plain text). |
| **Sticky Note** | “Document Assembly — Builds the full Google Doc content — original text plus all five tone versions with clear section headers. Also prepares the Slack preview message showing the Concise version.” |

---

### Block 1.4 — Binary Conversion & Google Drive Upload

**Overview**  
The assembled text is encoded as a Base64 binary file and uploaded to the configured Google Drive folder. The file is named with the project name and date.

**Nodes Involved**  
- 7. Code — Convert to Binary  
- 8. Google Drive — Upload Tone Doc  

#### 7. Code — Convert to Binary

| Attribute | Detail |
|-----------|--------|
| **Type** | `n8n-nodes-base.code` (v2) |
| **Technical Role** | Converts the plain‑text `docContent` into a binary buffer suitable for Google Drive upload |
| **Key Logic** | 1. Reads `docContent` from the incoming item. <br>2. Encodes it as UTF‑8, then Base64: `Buffer.from(textContent, 'utf-8').toString('base64')`. <br>3. Returns a JSON object with metadata (`docTitle`, `folderId`, `slackChannel`, `slackPreview`, `projectName`) plus a `binary` property containing `data` (Base64), `mimeType` “text/plain”, `fileName` `docTitle + '.txt'`, `fileExtension` “txt”. |
| **Connections** | ← 6. Code — Assemble Document Content (main 0) <br>→ 8. Google Drive — Upload Tone Doc (main) |
| **Edge Cases** | - If `docContent` is undefined, the conversion will produce an empty file. <br>- `Buffer` is available in n8n’s Node.js environment; no external library required. |
| **Sticky Note** | “Google Drive Upload — Converts the assembled text to binary format, then uploads it as a document to the configured Drive folder. File is named with project name and date.” |

#### 8. Google Drive — Upload Tone Doc

| Attribute | Detail |
|-----------|--------|
| **Type** | `n8n-nodes-base.googleDrive` (v3) |
| **Technical Role** | Uploads the binary `.txt` file to the specified Drive folder |
| **Parameters** | • `name` – expression `={{ $json.docTitle }}` <br>• `driveId` – mode “list”, value “My Drive” <br>• `folderId` – mode “id”, expression `={{ $json.folderId }}` <br>• `options` – none |
| **Credential** | Google Drive OAuth2 (must be authorised) |
| **Input Binary** | Uses the `data` binary property produced by node 7 |
| **Connections** | ← 7. Code — Convert to Binary (main) <br>Terminal node (no outgoing connections) |
| **Edge Cases** | - Invalid `folderId` yields a 404 or permission error. <br>- If the Google account lacks write access to the folder, the upload fails. |
| **Sticky Note** | Same as block 1.4 note. |

---

### Block 1.5 — Slack Notification

**Overview**  
A Slack message is posted with the project metadata and the Concise tone version, giving the team an instant preview while the full document is saved in Drive.

**Nodes Involved**  
- 9. Slack — Send Concise Preview  

#### 9. Slack — Send Concise Preview

| Attribute | Detail |
|-----------|--------|
| **Type** | `n8n-nodes-base.slack` (v2.2) |
| **Technical Role** | Posts a formatted message to a Slack channel |
| **Parameters** | • `text` – expression `={{ $json.slackPreview }}` <br>• `otherOptions.mrkdwn` – true (enables Slack markdown) <br>• `authentication` – “oAuth2” |
| **Channel** | Uses the `slackChannel` value from the config node (expected format `#channel-name`). In the node configuration the channel field should be set to `={{ $json.slackChannel }}` or configured statically. |
| **Credential** | Slack OAuth2 app (bot token with `chat:write` scope) |
| **Connections** | ← 6. Code — Assemble Document Content (main 1) <br>Terminal node |
| **Edge Cases** | - If the bot is not invited to the channel, Slack returns `channel_not_found`. <br>- Missing `#` prefix in `slackChannel` causes a post to a user DM instead of a channel. |
| **Sticky Note** | “Slack Preview — Posts the Concise tone version to your Slack channel as an instant team preview. Full five-version document is always in Google Drive.” |

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|-----------|-----------|-----------------|---------------|----------------|-------------|
| 1. Form — Content Tone Rewriter | formTrigger (v2.2) | Collects original text, content type, project name, and optional audience via web form | — | 2. Set — Config Values | “Form Input and Config — Form collects the original text, content type, project name, and target audience. Config stores Drive folder ID, Slack channel, company name, and builds the document title and run timestamp.” |
| 2. Set — Config Values | set (v3.4) | Normalises form fields, stores config placeholders, builds derived variables (docTitle, runDate) | 1. Form — Content Tone Rewriter | 3. AI Agent — Tone Rewriter | “Form Input and Config — Form collects the original text, content type, project name, and target audience. Config stores Drive folder ID, Slack channel, company name, and builds the document title and run timestamp.”<br>“⚠️ Edit This Node Before Activating — Replace three values: YOUR_GOOGLE_DRIVE_FOLDER_ID (from Drive folder URL after /folders/), YOUR-SLACK-CHANNEL (include the # symbol), and YOUR COMPANY NAME.” |
| 3. AI Agent — Tone Rewriter | @n8n/n8n-nodes-langchain.agent (v1.7) | Orchestrates LLM call with five‑tone prompt and structured output | 2. Set — Config Values (main), 4. OpenAI — GPT-4o-mini Model (ai_languageModel), 5. Parser — Structured Tone Output (ai_outputParser) | 6. Code — Assemble Document Content | “AI Tone Rewriting — GPT-4o-mini rewrites the original text in five tones simultaneously. Structured Output Parser enforces that all five versions are always returned: Professional, Casual, Persuasive, Concise, and Empathetic.” |
| 4. OpenAI — GPT-4o-mini Model | @n8n/n8n-nodes-langchain.lmChatOpenAi (v1.2) | Provides GPT‑4o‑mini as the language model backend | — | 3. AI Agent — Tone Rewriter (ai_languageModel) | “AI Tone Rewriting — GPT-4o-mini rewrites the original text in five tones simultaneously. Structured Output Parser enforces that all five versions are always returned: Professional, Casual, Persuasive, Concise, and Empathetic.” |
| 5. Parser — Structured Tone Output | @n8n/n8n-nodes-langchain.outputParserStructured (v1.3) | Enforces JSON schema with five required tone keys on the LLM response | — | 3. AI Agent — Tone Rewriter (ai_outputParser) | “AI Tone Rewriting — GPT-4o-mini rewrites the original text in five tones simultaneously. Structured Output Parser enforces that all five versions are always returned: Professional, Casual, Persuasive, Concise, and Empathetic.” |
| 6. Code — Assemble Document Content | code (v2) | Merges AI output and config into a full document string and a Slack preview message | 3. AI Agent — Tone Rewriter | 7. Code — Convert to Binary, 9. Slack — Send Concise Preview | “Document Assembly — Builds the full Google Doc content — original text plus all five tone versions with clear section headers. Also prepares the Slack preview message showing the Concise version.” |
| 7. Code — Convert to Binary | code (v2) | Encodes the assembled document text to Base64 binary for Drive upload | 6. Code — Assemble Document Content | 8. Google Drive — Upload Tone Doc | “Google Drive Upload — Converts the assembled text to binary format, then uploads it as a document to the configured Drive folder. File is named with project name and date.” |
| 8. Google Drive — Upload Tone Doc | googleDrive (v3) | Uploads the .txt file to the configured Drive folder | 7. Code — Convert to Binary | — | “Google Drive Upload — Converts the assembled text to binary format, then uploads it as a document to the configured Drive folder. File is named with project name and date.” |
| 9. Slack — Send Concise Preview | slack (v2.2) | Posts the Concise version preview to a Slack channel | 6. Code — Assemble Document Content | — | “Slack Preview — Posts the Concise tone version to your Slack channel as an instant team preview. Full five-version document is always in Google Drive.” |

*Sticky notes that are purely informational (Overview, Section headings) are reflected in the table only where they directly cover a node.*

---

## 4. Reproducing the Workflow from Scratch

Below is a numbered, step‑by‑step guide to recreate this workflow in a fresh n8n instance.

### 4.1 Create the Form Trigger

1. Add a **Form Trigger** node.  
   - **Type:** `n8n-nodes-base.formTrigger` (version 2.2)  
   - **Form Title:** “AI Content Tone Rewriter”  
   - **Form Description:** “Paste your text below. AI will rewrite it in 5 different tones and save all versions to Google Drive.”  
   - **Fields (add in order):**  
     1. **Your Original Text** – Type: Textarea, Required: true, Placeholder: “Paste your email, blog paragraph, social caption, proposal, or any text here…”  
     2. **Content Type** – Type: Dropdown, Required: true, Options: Email, Social Media Caption, Blog Paragraph, Client Proposal, Product Description, Other  
     3. **Your Name or Project Name** – Type: Text, Required: true, Placeholder: “e.g. Rahul — Incrementors Proposal”  
     4. **Target Audience** – Type: Text, Required: false, Placeholder: “e.g. C-suite executives, young professionals, general public”  
   - Click **Save**. Copy the generated **Form URL** for later use.

### 4.2 Create the Configuration (Set) Node

2. Add a **Set** node (type `n8n-nodes-base.set`, version 3.4).  
   - **Assignments (keep exact names):**  
     - `folderId` – String – “YOUR_GOOGLE_DRIVE_FOLDER_ID”  
     - `slackChannel` – String – “#YOUR-SLACK-CHANNEL”  
     - `companyName` – String – “YOUR COMPANY NAME”  
     - `originalText` – Expression – `={{ $json['Your Original Text'] }}`  
     - `contentType` – Expression – `={{ $json['Content Type'] }}`  
     - `projectName` – Expression – `={{ $json['Your Name or Project Name'] }}`  
     - `targetAudience` – Expression – `={{ $json['Target Audience'] || 'General audience' }}`  
     - `docTitle` – Expression – `=Tone Rewrites — {{ $json['Your Name or Project Name'] }} — {{ $now.toFormat('dd MMM yyyy') }}`  
     - `runDate` – Expression – `={{ $now.toFormat('dd MMM yyyy HH:mm') }}`  
   - **Options:** none (keep default).  
   - Click **Save**.  
   - ⚠️ **Important:** Replace the three placeholder strings before activating the workflow.

3. Connect the **Form Trigger** output to this **Set** node’s input (main connection).

### 4.3 Add the AI Agent and Its Sub‑Nodes

4. Add an **AI Agent** node (type `@n8n/n8n-nodes-langchain.agent`, version 1.7).  
   - **Prompt Type:** define (use the node’s text field).  
   - **Text (system prompt):** paste the full prompt shown in the workflow JSON (starting with “You are a professional content writer…”). It references `{{ $json.companyName }}`, `{{ $json.originalText }}`, `{{ $json.contentType }}`, and `{{ $json.targetAudience }}`.  
   - **hasOutputParser:** true.  
   - Click **Save**.

5. Add an **OpenAI Chat Model** node (type `@n8n/n8n-nodes-langchain.lmChatOpenAi`, version 1.2).  
   - **Model:** select `gpt-4o-mini` from the list.  
   - **Options:** `maxTokens`: 1500, `temperature`: 0.7.  
   - **Credential:** create or select an OpenAI API key credential.  
   - Click **Save**.  
   - Connect this node’s **ai_languageModel** output to the **AI Agent** node’s **ai_languageModel** input.

6. Add a **Structured Output Parser** node (type `@n8n/n8n-nodes-langchain.outputParserStructured`, version 1.3).  
   - **Schema Type:** manual.  
   - **Input Schema:** paste the JSON schema exactly as in the workflow (object with five required string properties: professional, casual, persuasive, concise, empathetic).  
   - Click **Save**.  
   - Connect this node’s **ai_outputParser** output to the **AI Agent** node’s **ai_outputParser** input.

7. Connect the **Set** node’s output to the **AI Agent** node’s main input.

### 4.4 Assemble the Document Content

8. Add a **Code** node (type `n8n-nodes-base.code`, version 2).  
   - **Language:** JavaScript (default).  
   - **Code:** paste the full `jsCode` from the workflow (it reads `$input.first().json.output`, builds `docContent` and `slackPreview`, returns an object with all needed fields).  
   - Click **Save**.  
   - Connect the **AI Agent** node’s main output to this **Code** node’s input.

### 4.5 Convert to Binary and Upload to Google Drive

9. Add a second **Code** node (type `n8n-nodes-base.code`, version 2).  
   - **Code:** paste the `jsCode` that converts `docContent` to Base64 and produces the binary payload (`fileName`, `mimeType`, `fileExtension`).  
   - Click **Save**.  
   - Connect the **Assemble Document Content** node’s **first** output (index 0) to this **Convert to Binary** node’s input.

10. Add a **Google Drive** node (type `n8n-nodes-base.googleDrive`, version 3).  
    - **Operation:** Upload (default when a binary is present).  
    - **Name:** expression `={{ $json.docTitle }}`.  
    - **Drive ID:** mode “list”, value “My Drive”.  
    - **Folder ID:** mode “id”, expression `={{ $json.folderId }}`.  
    - **Options:** none.  
    - **Credential:** create or select a Google Drive OAuth2 credential with Drive write access.  
    - Click **Save**.  
    - Connect the **Convert to Binary** node’s output to this **Google Drive** node’s input.

### 4.6 Send the Slack Preview

11. Add a **Slack** node (type `n8n-nodes-base.slack`, version 2.2).  
    - **Operation:** Send a message (default).  
    - **Channel:** expression `={{ $json.slackChannel }}` (or set manually to the desired channel).  
    - **Text:** expression `={{ $json.slackPreview }}`.  
    - **Other Options:** enable `mrkdwn` (true).  
    - **Authentication:** OAuth2.  
    - **Credential:** create or select a Slack OAuth2 app with `chat:write` scope.  
    - Click **Save**.  
    - Connect the **Assemble Document Content** node’s **second** output (index 1) to this **Slack** node’s input.

### 4.7 Finalise and Activate

12. Review all connections:  
    - Form → Set → AI Agent (with Model + Parser) → Code Assemble → (Convert to Binary → Google Drive) + (Slack).  
13. Replace the three placeholder values in the **Set** node with your actual Google Drive folder ID, Slack channel (including `#`), and company name.  
14. Ensure the **OpenAI**, **Google Drive**, and **Slack** credentials are all connected and valid.  
15. Click **Activate** in the top‑right corner of n8n.  
16. Open the **Form URL** from node 1 in a browser and test a submission.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|--------------|-----------------|
| This workflow was built by Incrementors. For custom versions or support, contact info@incrementors.com or visit https://incrementors.com/ | Support & credits |
| The OpenAI model `gpt-4o-mini` is used for cost efficiency. To increase quality, switch to `gpt-4o` in the model node (higher cost). | Model selection |
| Temperature is set to 0.7 for balanced creativity. Lower values (e.g., 0.3) produce more deterministic outputs; higher values (e.g., 0.9) increase variation but may reduce schema compliance. | Parameter tuning |
| To add more content types to the form, edit the dropdown options in node 1. | Customisation |
| To add a sixth tone, extend the prompt in node 3, add a new key to the schema in node 5, and update the assembly code in node 6. | Extending tones |
| The document is saved as a plain‑text `.txt` file. To produce a Google Doc or PDF, replace the upload node with the appropriate Google Docs or conversion node. | Output format |
| The Slack message uses mrkdwn formatting. Ensure your Slack app has the `chat:write` and `chat:write.public` scopes if posting to public channels. | Slack setup |
| The workflow’s form webhook ID is auto‑generated. After activation, always use the URL shown in node 1; do not hard‑code older URLs. | Webhook stability |