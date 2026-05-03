Structure AI meeting notes with GPT-4o-mini and save to Google Drive

https://n8nworkflows.xyz/workflows/structure-ai-meeting-notes-with-gpt-4o-mini-and-save-to-google-drive-14895


# Structure AI meeting notes with GPT-4o-mini and save to Google Drive

Now I have all the information I need. Let me create the comprehensive reference document. 1. Workflow Overview

This workflow, **"Structure AI meeting notes with GPT-4o-mini and save to Google Drive"**, is designed for agency teams, consultants, and project managers who need to automatically clean and structure raw, messy meeting notes into a professional 6-section document. A user submits their raw notes through a web form; the workflow extracts the form data, sends it to GPT-4o-mini for intelligent structuring, assembles the AI-returned sections into a clean plain-text document, and saves the resulting file directly to a specified Google Drive folder.

The logical flow is organized into four functional blocks:

| Block | Name | Purpose |
|-------|------|---------|
| 1 | **Form Input** | Collects meeting metadata (client name, date, attendees, raw notes, submitter name, Drive folder ID) via an n8n form trigger |
| 2 | **Data Preparation** | Maps all raw form field names into clean, predictable variable names and auto-generates a document title |
| 3 | **AI Notes Structuring** | Sends the cleaned data to an AI Agent powered by GPT-4o-mini, constrained by a structured output parser, which returns a JSON object with 6 sections: header, overview, key discussion points, decisions, action items, and next steps |
| 4 | **Document Assembly and Save** | Combines all 6 sections into a single formatted plain-text document and creates a new file in the user-specified Google Drive folder |

---

### 2. Block-by-Block Analysis

---

#### Block 1 — Form Input

**Overview:**  
This block is the single entry point of the workflow. It presents an n8n web form to the user, collecting all the information needed for downstream processing: client/project name, meeting date, attendees, raw meeting notes, the submitter's name, and the target Google Drive folder ID. Submitting the form triggers the entire workflow.

**Nodes Involved:**
- 1. Form — Meeting Notes Submission

**Node Details:**

**1. Form — Meeting Notes Submission**  
- **Type:** `n8n-nodes-base.formTrigger` (version 2.2)  
- **Technical Role:** Workflow trigger; renders an HTML form and emits form data on submission.  
- **Configuration:**
  - Form title: "Meeting Notes Submitter"
  - Form description: "Fill in your meeting details and raw notes. AI will clean and structure them automatically."
  - Six required fields, all with `requiredField: true`:
    1. **Client or Project Name** — text input, placeholder "e.g. Sadrian Plastic Surgery"
    2. **Meeting Date** — text input, placeholder "e.g. 08 April 2026"
    3. **Attendees** — text input, placeholder "e.g. Rahul, Deepak, Client"
    4. **Raw Meeting Notes** — textarea, placeholder "Paste your rough notes here. Can be bullet points, sentences, or messy text."
    5. **Your Name** — text input, placeholder "e.g. Rahul Dutt"
    6. **Google Drive Folder ID** — text input, placeholder "Paste your Google Drive folder ID here (from folder URL)"
  - No additional options configured.
- **Key Expressions / Variables:** None; outputs raw form field names as JSON keys matching `fieldLabel` values.
- **Input Connections:** None (entry node).  
- **Output Connections:** → `2. Set — Extract Form Fields` (main, index 0)  
- **Edge Cases / Failure Types:**
  - If any required field is left blank, the form will not submit (client-side validation).
  - The Google Drive Folder ID field is free-text; an invalid or inaccessible folder ID will not cause an error until Block 4.
  - No file upload capability; raw notes must be pasted as text.
  - Webhook ID is auto-generated; the form URL changes if the workflow is duplicated to a new instance.

---

#### Block 2 — Data Preparation

**Overview:**  
This block takes the raw form output (which uses human-readable field labels as JSON keys) and remaps everything into clean, predictable variable names (`clientName`, `meetingDate`, `attendees`, `rawNotes`, `submittedBy`, `folderId`). It also auto-generates a document title combining the client name and meeting date.

**Nodes Involved:**
- 2. Set — Extract Form Fields

**Node Details:**

**2. Set — Extract Form Fields**  
- **Type:** `n8n-nodes-base.set` (version 3.4)  
- **Technical Role:** Data transformation; renames and restructures form output into downstream-friendly variables.  
- **Configuration — Seven assignments:**

  | Assignment ID | Variable Name | Type | Expression |
  |---|---|---|---|
  | cfg-001 | `clientName` | string | `={{ $json['Client or Project Name'] }}` |
  | cfg-002 | `meetingDate` | string | `={{ $json['Meeting Date'] }}` |
  | cfg-003 | `attendees` | string | `={{ $json['Attendees'] }}` |
  | cfg-004 | `rawNotes` | string | `={{ $json['Raw Meeting Notes'] }}` |
  | cfg-005 | `submittedBy` | string | `={{ $json['Your Name'] }}` |
  | cfg-006 | `folderId` | string | `={{ $json['Google Drive Folder ID'] }}` |
  | cfg-007 | `docTitle` | string | `=Meeting Notes — {{ $json['Client or Project Name'] }} — {{ $json['Meeting Date'] }}` |

- **Key Expressions / Variables:**
  - `docTitle` is an expression (prefixed with `=`) that concatenates "Meeting Notes — ", the client name, " — ", and the meeting date. This becomes the file name for the Google Drive document.
  - All other assignments use `={{ ... }}` expression syntax to pull from the form output.
- **Input Connections:** ← `1. Form — Meeting Notes Submission` (main, index 0)  
- **Output Connections:** → `3. AI Agent — Structure Meeting Notes` (main, index 0)  
- **Edge Cases / Failure Types:**
  - If a form field label is renamed or removed in the form node, the corresponding expression here will return `undefined`, potentially causing downstream issues (e.g., `docTitle` containing "undefined").
  - No type coercion is performed; all values are strings.
  - The Set node passes through all original fields in addition to the new ones, but downstream nodes reference only the new variable names.

---

#### Block 3 — AI Notes Structuring

**Overview:**  
This is the core intelligence block. An AI Agent receives the cleaned variables from Block 2 and constructs a detailed prompt instructing GPT-4o-mini to convert raw meeting notes into a structured 6-section JSON. The OpenAI chat model and a structured output parser are wired into the agent as sub-nodes, ensuring the model returns a valid, parseable JSON conforming to a strict schema.

**Nodes Involved:**
- 3. AI Agent — Structure Meeting Notes
- 4. OpenAI — GPT-4o-mini Model
- 5. Parser — Structured Notes Output

**Node Details:**

**3. AI Agent — Structure Meeting Notes**  
- **Type:** `@n8n/n8n-nodes-langchain.agent` (version 1.7)  
- **Technical Role:** LangChain-based AI agent that sends a constructed prompt to the connected LLM and returns structured output via an output parser.  
- **Configuration:**
  - `promptType`: "define" — the prompt is defined inline (not from a previous node).
  - `hasOutputParser`: true — signals that an output parser sub-node is connected.
  - `text` (the full prompt template) — constructed as an expression string that embeds variables from the incoming item:

    ```
    You are a professional meeting notes formatter for a digital marketing agency.

    Your job is to take raw messy meeting notes and convert them into a clean structured document.

    MEETING DETAILS:
    Client or Project: {{ $json.clientName }}
    Date: {{ $json.meetingDate }}
    Attendees: {{ $json.attendees }}
    Submitted by: {{ $json.submittedBy }}

    RAW NOTES:
    {{ $json.rawNotes }}

    You must return a JSON object with exactly these 6 fields:

    overview — 2 to 3 plain sentences summarizing what the meeting was about.

    keyDiscussionPoints — a plain text list of major topics discussed. Number each point. Use full sentences. Minimum 3 points maximum 8 points. Separate each point with a newline.

    decisionsMade — a plain text list of decisions finalized in the meeting. If none are clear write: No formal decisions recorded.

    actionItems — a plain text list of tasks and follow-ups. For each item write the task description then a pipe symbol then the owner then a pipe symbol then the deadline. If owner or deadline is not mentioned write Not assigned or Not specified. Separate each item with a newline.

    nextSteps — 2 to 3 plain sentences about what happens next after this meeting.

    headerBlock — a plain text block with exactly these 4 lines:
    Client: [client name]
    Date: [meeting date]
    Attendees: [attendees]
    Prepared by: [submitted by name]

    RULES:
    - Plain text only inside every field. No markdown. No asterisks. No hashtags.
    - Do not add anything not present in the raw notes.
    - If a section has no data from notes write: Not discussed in this meeting.
    ```

- **Key Expressions / Variables:**
  - `{{ $json.clientName }}`, `{{ $json.meetingDate }}`, `{{ $json.attendees }}`, `{{ $json.submittedBy }}`, `{{ $json.rawNotes }}` — all reference the cleaned variables from the Set node.
- **Input Connections (main):** ← `2. Set — Extract Form Fields` (main, index 0)  
- **Sub-node Connections:**
  - `ai_languageModel` ← `4. OpenAI — GPT-4o-mini Model` (ai_languageModel, index 0)
  - `ai_outputParser` ← `5. Parser — Structured Notes Output` (ai_outputParser, index 0)
- **Output Connections:** → `6. Code — Assemble Full Document` (main, index 0)  
- **Edge Cases / Failure Types:**
  - If `rawNotes` is empty or minimal, the model may produce generic or repetitive content.
  - If the model fails to produce valid JSON matching the schema, the output parser will throw an error. The low temperature (0.3) reduces but does not eliminate this risk.
  - Token limit: `maxTokens` is set to 1200; very long raw notes may cause truncation if the combined prompt + response exceeds the model's context window (128k for gpt-4o-mini, so unlikely in practice).
  - OpenAI API key must be valid and have sufficient quota.
  - Rate limiting or OpenAI service outages will cause this node to fail.
  - The `$json` reference assumes a single item flow; multi-item batches are not handled.

---

**4. OpenAI — GPT-4o-mini Model**  
- **Type:** `@n8n/n8n-nodes-langchain.lmChatOpenAi` (version 1.2)  
- **Technical Role:** LLM sub-node providing the chat model for the AI Agent.  
- **Configuration:**
  - Model: `gpt-4o-mini` (selected from list mode)
  - Temperature: 0.3 — low creativity for consistent, factual structuring
  - Max tokens: 1200 — limits response length to save costs and encourage concise output
- **Key Expressions / Variables:** None  
- **Input Connections:** None (receives requests from the AI Agent at runtime)  
- **Output Connections:** → `3. AI Agent — Structure Meeting Notes` (ai_languageModel, index 0)  
- **Edge Cases / Failure Types:**
  - Requires a valid OpenAI API credential configured on this node.
  - If the API key has insufficient credits or is revoked, the agent will fail with an authentication/billing error.
  - Model name `gpt-4o-mini` must be available on the connected OpenAI account; deprecated models would cause failure.
  - `maxTokens: 1200` may truncate responses for meetings with very extensive notes; the output would be incomplete JSON, likely causing the parser to fail.

---

**5. Parser — Structured Notes Output**  
- **Type:** `@n8n/n8n-nodes-langchain.outputParserStructured` (version 1.3)  
- **Technical Role:** Constrained output parser that enforces a JSON schema on the LLM response, ensuring the output is a valid object with exactly the 6 required fields.  
- **Configuration:**
  - `schemaType`: "manual" — schema is defined inline (not fetched from a URL or external source).
  - `inputSchema` — a manually defined JSON Schema:

    ```json
    {
      "type": "object",
      "properties": {
        "headerBlock": {
          "type": "string",
          "description": "Plain text block with Client, Date, Attendees, Prepared by lines"
        },
        "overview": {
          "type": "string",
          "description": "2 to 3 sentence plain text summary of the meeting"
        },
        "keyDiscussionPoints": {
          "type": "string",
          "description": "Numbered plain text list of major topics discussed, each on a new line"
        },
        "decisionsMade": {
          "type": "string",
          "description": "Plain text list of decisions. If none write: No formal decisions recorded."
        },
        "actionItems": {
          "type": "string",
          "description": "Plain text list of tasks. Format: Task | Owner | Deadline. Each item on new line."
        },
        "nextSteps": {
          "type": "string",
          "description": "2 to 3 sentences about what happens after this meeting"
        }
      },
      "required": ["headerBlock", "overview", "keyDiscussionPoints", "decisionsMade", "actionItems", "nextSteps"]
    }
    ```

- **Key Expressions / Variables:** None (schema is static)  
- **Input Connections:** None (receives parsing directives from the AI Agent at runtime)  
- **Output Connections:** → `3. AI Agent — Structure Meeting Notes` (ai_outputParser, index 0)  
- **Edge Cases / Failure Types:**
  - If the LLM output does not conform to this schema (missing field, wrong type), the parser will throw a validation error and the workflow will stop.
  - The `required` array enforces all 6 fields must be present; the prompt's rules instruct the model to use fallback text like "Not discussed in this meeting" rather than omit a field, which should prevent missing-field errors.
  - All fields are type `string`; if the model returns an array for `actionItems` instead of a newline-separated string, parsing will fail.

---

#### Block 4 — Document Assembly and Save

**Overview:**  
This block takes the structured JSON output from the AI Agent and assembles it into a single, human-readable plain-text document with clearly labeled sections. It also retrieves the document title and folder ID from the earlier Set node. The assembled document is then saved as a new file in the specified Google Drive folder.

**Nodes Involved:**
- 6. Code — Assemble Full Document
- 7. Google Drive — Save Notes Document

**Node Details:**

**6. Code — Assemble Full Document**  
- **Type:** `n8n-nodes-base.code` (version 2)  
- **Technical Role:** Custom JavaScript execution; converts the 6-field structured JSON into a formatted plain-text string and passes the document title and folder ID forward.  
- **Configuration:**
  - `jsCode` — inline JavaScript. Full logic:

    ```javascript
    const out = $input.first().json.output;

    const headerBlock    = out.headerBlock         || 'Client: N/A\nDate: N/A\nAttendees: N/A\nPrepared by: N/A';
    const overview       = out.overview             || 'Not available.';
    const keyPoints      = out.keyDiscussionPoints  || 'Not discussed in this meeting.';
    const decisions      = out.decisionsMade        || 'No formal decisions recorded.';
    const actionItems    = out.actionItems          || 'No action items recorded.';
    const nextSteps      = out.nextSteps            || 'Not discussed in this meeting.';

    const fullDoc = `MEETING SUMMARY
    ${headerBlock}

    OVERVIEW
    ${overview}

    KEY DISCUSSION POINTS
    ${keyPoints}

    DECISIONS MADE
    ${decisions}

    ACTION ITEMS
    ${actionItems}

    NEXT STEPS
    ${nextSteps}

    ---
    This document was auto-generated by n8n + GPT-4o-mini`;

    return [{
      json: {
        formattedNotes: fullDoc,
        docTitle: $('2. Set — Extract Form Fields').item.json.docTitle,
        folderId: $('2. Set — Extract Form Fields').item.json.folderId
      }
    }];
    ```

- **Key Expressions / Variables:**
  - `$input.first().json.output` — accesses the structured output from the AI Agent. The agent's parsed result is nested under an `output` key.
  - Fallback strings are provided for every field in case a value is missing or undefined.
  - `$('2. Set — Extract Form Fields').item.json.docTitle` — references the `docTitle` variable from the Set node by node name.
  - `$('2. Set — Extract Form Fields').item.json.folderId` — references the `folderId` variable from the Set node by node name.
- **Output Structure:**
  - `formattedNotes` (string): the complete plain-text document
  - `docTitle` (string): the auto-generated document title
  - `folderId` (string): the Google Drive folder ID from the form
- **Input Connections:** ← `3. AI Agent — Structure Meeting Notes` (main, index 0)  
- **Output Connections:** → `7. Google Drive — Save Notes Document` (main, index 0)  
- **Edge Cases / Failure Types:**
  - If the AI Agent output structure changes (e.g., `output` key is renamed), `out` will be undefined and all fields will use fallbacks, producing a document with "N/A" values.
  - Cross-node references using `$('2. Set — Extract Form Fields')` assume that node retains its exact name; renaming the Set node breaks these references.
  - If multiple items are passed through the workflow (batch scenario), `$input.first()` only processes the first item.
  - JavaScript runtime errors (e.g., accessing a property on undefined if the entire `json` structure is unexpected) will cause the node to fail.

---

**7. Google Drive — Save Notes Document**  
- **Type:** `n8n-nodes-base.googleDrive` (version 3)  
- **Technical Role:** Creates a new file in Google Drive using the assembled document content.  
- **Configuration:**
  - Operation: `create` — creates a new file.
  - No other parameters are explicitly set in the workflow JSON (folder ID, file name, and content are expected to be mapped from incoming data via the node's default field mapping or via the UI's field configuration which isn't fully captured in the exported JSON). Based on the incoming data from the Code node, the expected mapping is:
    - **File name** ← `docTitle` (from Code node output)
    - **Folder** ← `folderId` (from Code node output)
    - **File content** ← `formattedNotes` (from Code node output)
- **Key Expressions / Variables:** The node is expected to reference `{{ $json.docTitle }}`, `{{ $json.folderId }}`, and `{{ $json.formattedNotes }}` from the incoming Code node output. Exact mapping depends on the Google Drive node UI configuration which defaults to reading from the current item's JSON.  
- **Input Connections:** ← `6. Code — Assemble Full Document` (main, index 0)  
- **Output Connections:** None (terminal node)  
- **Version-Specific Requirements:** Google Drive node v3 uses OAuth2 authentication and requires the Google Drive API to be enabled.  
- **Edge Cases / Failure Types:**
  - **No credential connected** — This is explicitly flagged by the workflow's sticky note. The node will fail with an authentication error if no Google Drive OAuth2 credential is configured.
  - **Invalid folder ID** — If the provided folder ID does not exist or the authenticated user lacks write access, the API will return a 404 or 403 error.
  - **File name conflicts** — If a file with the same name already exists, Google Drive will create a new revision or a separate file depending on the node's version behavior.
  - **Content size limits** — Google Drive API supports large files, but very lengthy meeting notes are unlikely to hit limits.
  - **OAuth token expiration** — If the OAuth2 token expires or is revoked, the node will fail. n8n typically handles token refresh automatically if the credential was set up with offline access.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Overview | n8n-nodes-base.stickyNote | General documentation of workflow purpose, how it works, and setup steps | — | — | ## AI Meeting Notes Structurer — GPT-4o-mini + Google Drive — For agency teams, consultants, and project managers who want to automatically clean and structure messy meeting notes. Submit raw notes via a web form — the workflow extracts all details, passes them to GPT-4o-mini which writes a structured 6-section document, assembles it into a clean plain-text file, and saves it directly to your Google Drive folder. / How it works: 1. Form — Meeting Notes Submission collects client name, date, attendees, raw notes, your name, and a Drive folder ID. 2. Set — Extract Form Fields maps all form inputs into clean named variables and builds the document title. 3. AI Agent — Structure Meeting Notes uses GPT-4o-mini to convert raw notes into 6 structured sections. 6. Code — Assemble Full Document combines all 6 sections into one complete plain-text document. 7. Google Drive — Save Notes Document creates a new file in your specified Drive folder. / Set up steps: 1. In 4. OpenAI — GPT-4o-mini Model — connect your OpenAI API credential. 2. In 7. Google Drive — Save Notes Document — connect your Google Drive OAuth2 credential. 3. Activate the workflow and copy the Form URL from node 1. 4. Open the Form URL in a browser, fill in all fields, and submit. 5. To find your Google Drive Folder ID — open the folder in your browser and copy the ID from the URL after /folders/ |
| Section — Form Input | n8n-nodes-base.stickyNote | Documentation for the form input section | — | — | ## Form Input — User submits client name, meeting date, attendees, raw notes, their name, and a Google Drive folder ID. Submitting the form triggers the workflow. |
| 1. Form — Meeting Notes Submission | n8n-nodes-base.formTrigger | Workflow entry point; collects meeting details via web form | — | 2. Set — Extract Form Fields | ## Form Input — User submits client name, meeting date, attendees, raw notes, their name, and a Google Drive folder ID. Submitting the form triggers the workflow. |
| Section — Data Preparation | n8n-nodes-base.stickyNote | Documentation for the data preparation section | — | — | ## Data Preparation — Maps all form inputs into clean named variables and auto-builds the document title from client name and meeting date. |
| 2. Set — Extract Form Fields | n8n-nodes-base.set | Remaps form fields to clean variable names; generates docTitle | 1. Form — Meeting Notes Submission | 3. AI Agent — Structure Meeting Notes | ## Data Preparation — Maps all form inputs into clean named variables and auto-builds the document title from client name and meeting date. |
| Section — AI Notes Structuring | n8n-nodes-base.stickyNote | Documentation for the AI structuring section | — | — | ## AI Notes Structuring — GPT-4o-mini reads the raw meeting notes and returns a structured 6-section JSON: header, overview, key discussion points, decisions, action items, and next steps. |
| 3. AI Agent — Structure Meeting Notes | @n8n/n8n-nodes-langchain.agent | AI agent that structures raw notes into 6-section JSON using GPT-4o-mini | 2. Set — Extract Form Fields (main); 4. OpenAI — GPT-4o-mini Model (ai_languageModel); 5. Parser — Structured Notes Output (ai_outputParser) | 6. Code — Assemble Full Document | ## AI Notes Structuring — GPT-4o-mini reads the raw meeting notes and returns a structured 6-section JSON: header, overview, key discussion points, decisions, action items, and next steps. |
| 4. OpenAI — GPT-4o-mini Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM sub-node providing GPT-4o-mini as the chat model for the AI Agent | — | 3. AI Agent — Structure Meeting Notes | ## AI Notes Structuring — GPT-4o-mini reads the raw meeting notes and returns a structured 6-section JSON: header, overview, key discussion points, decisions, action items, and next steps. |
| 5. Parser — Structured Notes Output | @n8n/n8n-nodes-langchain.outputParserStructured | Output parser enforcing the 6-field JSON schema on LLM response | — | 3. AI Agent — Structure Meeting Notes | ## AI Notes Structuring — GPT-4o-mini reads the raw meeting notes and returns a structured 6-section JSON: header, overview, key discussion points, decisions, action items, and next steps. |
| Section — Document Assembly and Save | n8n-nodes-base.stickyNote | Documentation for the document assembly and save section | — | — | ## Document Assembly and Save — Assembles all 6 sections into one clean plain-text document, then creates a new file in the Google Drive folder specified in the form. |
| 6. Code — Assemble Full Document | n8n-nodes-base.code | Assembles the 6 AI sections into a single formatted plain-text document; passes docTitle and folderId forward | 3. AI Agent — Structure Meeting Notes | 7. Google Drive — Save Notes Document | ## Document Assembly and Save — Assembles all 6 sections into one clean plain-text document, then creates a new file in the Google Drive folder specified in the form. |
| Note — Google Drive Credential Required | n8n-nodes-base.stickyNote | Warning about missing Google Drive credential | — | — | ## ⚠️ Google Drive Credential Required — This node has no credential connected. Connect your Google Drive OAuth2 credential before activating. Without it the workflow will fail at the final step and the document will not be saved. |
| 7. Google Drive — Save Notes Document | n8n-nodes-base.googleDrive | Creates a new file in Google Drive with the assembled meeting notes | 6. Code — Assemble Full Document | — | ## Document Assembly and Save — Assembles all 6 sections into one clean plain-text document, then creates a new file in the Google Drive folder specified in the form. / ## ⚠️ Google Drive Credential Required — This node has no credential connected. Connect your Google Drive OAuth2 credential before activating. Without it the workflow will fail at the final step and the document will not be saved. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in your n8n instance.

2. **Add the Form Trigger node:**
   - Click **Add Node** → search for "Form Trigger" → select `Form Trigger`.
   - Rename it to **"1. Form — Meeting Notes Submission"**.
   - Set **Form Title** to: `Meeting Notes Submitter`.
   - Set **Form Description** to: `Fill in your meeting details and raw notes. AI will clean and structure them automatically.`
   - Add six form fields (all with **Required** enabled):
     1. Field label: `Client or Project Name`, placeholder: `e.g. Sadrian Plastic Surgery`
     2. Field label: `Meeting Date`, placeholder: `e.g. 08 April 2026`
     3. Field label: `Attendees`, placeholder: `e.g. Rahul, Deepak, Client`
     4. Field label: `Raw Meeting Notes`, field type: `Textarea`, placeholder: `Paste your rough notes here. Can be bullet points, sentences, or messy text.`
     5. Field label: `Your Name`, placeholder: `e.g. Rahul Dutt`
     6. Field label: `Google Drive Folder ID`, placeholder: `Paste your Google Drive folder ID here (from folder URL)`

3. **Add the Set node:**
   - Click **Add Node** → search for "Set" → select `Set` (version 3+).
   - Rename it to **"2. Set — Extract Form Fields"**.
   - Add seven assignments:
     1. Name: `clientName`, type: String, value: `={{ $json['Client or Project Name'] }}`
     2. Name: `meetingDate`, type: String, value: `={{ $json['Meeting Date'] }}`
     3. Name: `attendees`, type: String, value: `={{ $json['Attendees'] }}`
     4. Name: `rawNotes`, type: String, value: `={{ $json['Raw Meeting Notes'] }}`
     5. Name: `submittedBy`, type: String, value: `={{ $json['Your Name'] }}`
     6. Name: `folderId`, type: String, value: `={{ $json['Google Drive Folder ID'] }}`
     7. Name: `docTitle`, type: String, value: `=Meeting Notes — {{ $json['Client or Project Name'] }} — {{ $json['Meeting Date'] }}`
   - Connect the output of node 1 to the input of node 2.

4. **Add the AI Agent node:**
   - Click **Add Node** → search for "AI Agent" → select `AI Agent` (from the LangChain collection, version 1.7+).
   - Rename it to **"3. AI Agent — Structure Meeting Notes"**.
   - Set **Prompt Type** to: `Define`.
   - Enable **Has Output Parser** (toggle on).
   - In the **Text / Prompt** field, paste the full prompt (see the exact text in Block 3 above under node 3). The prompt uses these expression placeholders:
     - `{{ $json.clientName }}`, `{{ $json.meetingDate }}`, `{{ $json.attendees }}`, `{{ $json.submittedBy }}`, `{{ $json.rawNotes }}`
   - Connect the main output of node 2 to the main input of node 3.

5. **Add the OpenAI Chat Model sub-node:**
   - Click **Add Node** → search for "OpenAI Chat Model" → select `OpenAI Chat Model` (from the LangChain collection, version 1.2+).
   - Rename it to **"4. OpenAI — GPT-4o-mini Model"**.
   - Set **Model** to: `gpt-4o-mini` (use list mode to select).
   - Under **Options**, set:
     - **Temperature**: `0.3`
     - **Max Tokens**: `1200`
   - **Credential:** Create or select an OpenAI API credential (API key from https://platform.openai.com/api-keys).
   - Connect this node's output to the **Language Model** input (ai_languageModel) of node 3.

6. **Add the Structured Output Parser sub-node:**
   - Click **Add Node** → search for "Structured Output Parser" → select `Structured Output Parser` (from the LangChain collection, version 1.3+).
   - Rename it to **"5. Parser — Structured Notes Output"**.
   - Set **Schema Type** to: `Manual`.
   - In the **Input Schema** field, paste the JSON Schema object (see the exact schema in Block 3 above under node 5). It defines six required string properties: `headerBlock`, `overview`, `keyDiscussionPoints`, `decisionsMade`, `actionItems`, `nextSteps`.
   - Connect this node's output to the **Output Parser** input (ai_outputParser) of node 3.

7. **Add the Code node:**
   - Click **Add Node** → search for "Code" → select `Code` (version 2).
   - Rename it to **"6. Code — Assemble Full Document"**.
   - Set **Language** to: JavaScript (default).
   - Paste the full JavaScript code (see Block 4 above under node 6) into the code editor. The code:
     - Reads `$input.first().json.output` for the 6 AI-structured fields
     - Provides fallback values for each field
     - Assembles a formatted plain-text document with section headers
     - Returns `formattedNotes`, `docTitle`, and `folderId` (the latter two fetched from the Set node by name)
   - Connect the main output of node 3 to the main input of node 6.

8. **Add the Google Drive node:**
   - Click **Add Node** → search for "Google Drive" → select `Google Drive` (version 3).
   - Rename it to **"7. Google Drive — Save Notes Document"**.
   - Set **Operation** to: `Create`.
   - Configure the remaining fields to map from the incoming Code node data:
     - **File Name**: `={{ $json.docTitle }}`
     - **Folder**: `={{ $json.folderId }}` (select "By ID" mode if prompted)
     - **File Content**: `={{ $json.formattedNotes }}`
   - **Credential:** Create or select a Google Drive OAuth2 credential:
     - In n8n, go to Credentials → New Credential → Google Drive OAuth2 API.
     - Follow the OAuth2 consent flow. Ensure the Google Cloud project has the Google Drive API enabled and the scope includes `https://www.googleapis.com/auth/drive.file`.
   - Connect the main output of node 6 to the main input of node 7.

9. **Verify the complete connection chain:**
   - `1. Form` → `2. Set` → `3. AI Agent` → `6. Code` → `7. Google Drive`
   - `4. OpenAI Model` → `3. AI Agent` (ai_languageModel)
   - `5. Parser` → `3. AI Agent` (ai_outputParser)

10. **Activate the workflow** using the toggle in the top-right corner of the n8n editor.

11. **Copy the Form URL** from node 1 (visible in the node's panel after activation).

12. **Test the workflow** by opening the Form URL in a browser, filling in all six fields, and submitting. Verify that:
    - The AI Agent returns valid JSON with all 6 sections.
    - The Code node assembles a readable plain-text document.
    - A new file appears in the specified Google Drive folder with the correct title and content.

13. **To find your Google Drive Folder ID:** Open the target folder in Google Drive in your browser. The URL will be of the form `https://drive.google.com/drive/folders/FOLDER_ID` — copy the `FOLDER_ID` portion.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| This workflow uses the LangChain integration nodes (`@n8n/n8n-nodes-langchain`), which require n8n version 1.0+ with the LangChain nodes package installed. | Ensure your n8n instance is up to date. |
| The Google Drive node currently has no credential connected in the exported workflow. It must be configured before activation or the workflow will fail at the final step. | Flagged by the "⚠️ Google Drive Credential Required" sticky note in the workflow canvas. |
| The Google Drive OAuth2 credential requires the Google Drive API to be enabled in your Google Cloud project and the OAuth consent screen to be configured. | Google Cloud Console: https://console.cloud.google.com/ |
| The OpenAI API credential requires a valid API key with access to the `gpt-4o-mini` model. | OpenAI API keys: https://platform.openai.com/api-keys |
| The form trigger generates a unique webhook ID. If the workflow is imported or duplicated, a new webhook ID is assigned and the form URL changes. | Always copy the form URL from the node after activation. |
| The prompt instructs the AI to produce plain text only (no markdown). If the model occasionally produces markdown, the output will still be valid JSON strings but may contain asterisks or hashtags. | Monitor output quality; adjust the prompt rules if needed. |
| The `docTitle` expression uses an em dash (—) character. Ensure your system supports UTF-8 encoding to avoid file name issues on Google Drive. | Standard for Google Drive; unlikely to cause issues. |
| The Code node references the Set node by its exact name. If you rename the Set node, you must also update the two `$('2. Set — Extract Form Fields')` references in the Code node's JavaScript. | Critical for workflow integrity after renaming. |
| The workflow is designed for single-item processing (one form submission at a time). If multiple submissions arrive concurrently, n8n will process them as separate executions; each will run independently. | No special batch handling is needed. |
| The assembled document ends with a footer line: "This document was auto-generated by n8n + GPT-4o-mini". This is hardcoded in the Code node. | Remove or customize the footer by editing the template string in the Code node. |