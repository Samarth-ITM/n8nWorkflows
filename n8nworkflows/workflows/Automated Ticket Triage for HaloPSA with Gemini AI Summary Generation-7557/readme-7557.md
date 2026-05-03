Automated Ticket Triage for HaloPSA with Gemini AI Summary Generation

https://n8nworkflows.xyz/workflows/automated-ticket-triage-for-halopsa-with-gemini-ai-summary-generation-7557


# Automated Ticket Triage for HaloPSA with Gemini AI Summary Generation

### 1. Workflow Overview

This workflow automates ticket triage for HaloPSA by generating AI-powered summaries and actionable notes using Gemini AI. It is designed for MSPs (Managed Service Providers) to streamline support ticket handling by extracting key ticket details, summarizing and suggesting next steps through an LLM (Large Language Model), then posting a formatted private note back into HaloPSA.

The workflow logic is divided into the following blocks:

- **1.1 Input Reception:** Receives new ticket data from HaloPSA via a webhook.
- **1.2 Optional Filtering (Guard):** Optionally filters out tickets from specific teams to exclude them from processing.
- **1.3 Ticket Data Extraction:** Extracts relevant ticket fields (ID, summary, details) from the webhook payload.
- **1.4 AI Prompt Construction:** Builds a detailed prompt tailored for an LLM to generate a structured summary and recommendations.
- **1.5 AI Processing:** Sends the prompt to Gemini AI (or alternative LLM) and receives the AI-generated JSON output.
- **1.6 AI Output Parsing:** Cleans and parses the AI response into discrete JSON fields.
- **1.7 HTML Note Creation:** Formats the AI output into a branded HTML note suitable for HaloPSA.
- **1.8 Payload Wrapping:** Wraps the note and metadata into a JSON payload for HaloPSA’s Actions API.
- **1.9 Posting Back to HaloPSA:** Sends the constructed note to HaloPSA via HTTP request to create a private ticket note.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block sets up a webhook endpoint for HaloPSA to send new ticket data via HTTP POST. It is the entry point of the workflow.

- **Nodes Involved:**  
  - Note: Webhook (sticky note)  
  - Webhook

- **Node Details:**

  - **Note: Webhook**  
    - Type: Sticky Note  
    - Role: Documentation to instruct users on configuring the webhook path and payload in HaloPSA.  
    - Key info: Use POST method; set unique path; payload must include `ticket` and optionally `team` or `team_id`.  
    - Inputs: None  
    - Outputs: None  
    - Failure modes: None (informational only)

  - **Webhook**  
    - Type: Webhook (n8n core node)  
    - Role: Receives incoming HTTP POST requests from HaloPSA containing new ticket data.  
    - Configuration:  
      - HTTP Method: POST  
      - Path: `WEBHOOK_PATH` (to be replaced by user with unique path)  
    - Inputs: External HTTP request  
    - Outputs: JSON payload containing ticket info  
    - Failure modes:  
      - Incorrect path or method will cause webhook not to trigger.  
      - Payload missing required fields may cause downstream errors.

#### 2.2 Optional Filtering (Guard)

- **Overview:**  
  Filters out tickets from specified teams to avoid processing unwanted tickets (e.g., sales team).

- **Nodes Involved:**  
  - Note: Guard (sticky note)  
  - Guard (Code node)

- **Node Details:**

  - **Note: Guard**  
    - Type: Sticky Note  
    - Role: Explains that filtering can be customized or removed if not needed.  
    - Inputs/Outputs: None

  - **Guard**  
    - Type: Code (JavaScript)  
    - Role: Checks ticket’s team name or ID and stops workflow if it matches excluded teams.  
    - Key logic:  
      - Extracts `team` name or `team_id` from webhook payload.  
      - Compares against hardcoded filters (default: `sales` or team ID `6`).  
      - Returns empty array to stop workflow if filtered.  
    - Inputs: Output of Webhook  
    - Outputs: Pass-through if not filtered; empty to halt if filtered.  
    - Failure modes:  
      - Missing `team` or `team_id` fields handled gracefully (no halt).  
      - Malformed payloads may cause code errors.

#### 2.3 Ticket Data Extraction

- **Overview:**  
  Extracts essential ticket fields and formats a summary card string for later use.

- **Nodes Involved:**  
  - Note: Extract (sticky note)  
  - Extract Ticket (Code node)

- **Node Details:**

  - **Note: Extract**  
    - Type: Sticky Note  
    - Role: Explains which fields are extracted and that key names can be changed if payload structure differs.  
    - Inputs/Outputs: None

  - **Extract Ticket**  
    - Type: Code (JavaScript)  
    - Role: Parses the ticket object from the webhook, builds a summary string including ticket ID, summary, and details.  
    - Key expressions:  
      - Throws error if `ticket` object missing.  
      - Outputs: `ticket_id`, `summary_card` (formatted string), `details`, `subject`.  
    - Inputs: Output of Guard  
    - Outputs: JSON with extracted fields  
    - Failure modes:  
      - Missing ticket object triggers error halting workflow.

#### 2.4 AI Prompt Construction

- **Overview:**  
  Constructs a prompt string targeted at the AI model to generate structured JSON including summary and suggestions.

- **Nodes Involved:**  
  - Note: Prompt (sticky note)  
  - Build AI Prompt (Code node)

- **Node Details:**

  - **Note: Prompt**  
    - Type: Sticky Note  
    - Role: Instructs user to edit prompt wording and list of MSP tools used.  
    - Inputs/Outputs: None

  - **Build AI Prompt**  
    - Type: Code (JavaScript)  
    - Role: Builds a detailed prompt embedding ticket info and system context.  
    - Key variables: `subject`, `details`, `ticket_id`.  
    - Prompt includes: role definition, systems used (NinjaOne, Microsoft 365, CIPP), and ticket info.  
    - Inputs: Output of Extract Ticket  
    - Outputs: JSON with `prompt` string and `ticket_id`.  
    - Failure modes:  
      - Missing input fields may produce incomplete prompts.

#### 2.5 AI Processing

- **Overview:**  
  Sends the constructed prompt to an LLM agent (configured for Google Gemini) and receives the AI-generated response.

- **Nodes Involved:**  
  - Note: AI Agent (sticky note)  
  - AI Agent (LangChain node)  
  - Note: LLM Model (sticky note)  
  - Google Gemini Chat Model (LLM node)

- **Node Details:**

  - **Note: AI Agent**  
    - Type: Sticky Note  
    - Role: Instructions for connecting an LLM node; mentions Gemini and alternatives; credentials set in model node.  
    - Inputs/Outputs: None

  - **AI Agent**  
    - Type: LangChain Agent node  
    - Role: Receives prompt text and routes it to the connected LLM model node.  
    - Configuration:  
      - Text input expression: `{{$json.prompt}}`  
      - Prompt type: define (custom prompt)  
    - Inputs: Output of Build AI Prompt  
    - Outputs: AI response (expected JSON string)  
    - Failure modes:  
      - API errors, timeouts, malformed or missing prompt cause failure.  
      - Model credentials misconfigured.

  - **Note: LLM Model**  
    - Type: Sticky Note  
    - Role: Explains that API credentials must be set here; example with Google Gemini Chat Model.  
    - Inputs/Outputs: None

  - **Google Gemini Chat Model**  
    - Type: LangChain Google Gemini Chat Model node  
    - Role: The actual LLM endpoint node that communicates with Google Gemini API.  
    - Configuration: Credentials required (Google API key or OAuth)  
    - Inputs: Connected to AI Agent as language model  
    - Outputs: AI generated text  
    - Failure modes:  
      - Credential issues, quota exceeded, network errors.

#### 2.6 AI Output Parsing

- **Overview:**  
  Cleans the AI response text of markdown JSON fences and parses it into usable JSON fields.

- **Nodes Involved:**  
  - Note: Parse (sticky note)  
  - Parse AI JSON (Code node)

- **Node Details:**

  - **Note: Parse**  
    - Type: Sticky Note  
    - Role: Explains removing markdown code fences and parsing JSON fields (summary, next step, troubleshooting).  
    - Inputs/Outputs: None

  - **Parse AI JSON**  
    - Type: Code (JavaScript)  
    - Role:  
      - Removes triple backtick fences from AI output if present.  
      - Parses JSON string into object.  
      - Returns fields: `summary`, `next_step`, `troubleshooting` (HTML), `ticket_id`.  
    - Inputs: Output of AI Agent  
    - Outputs: Clean JSON with extracted fields  
    - Failure modes:  
      - JSON parse errors if AI output is malformed or incomplete.  
      - Missing expected keys.

#### 2.7 HTML Note Creation

- **Overview:**  
  Generates a branded HTML note incorporating the AI summary and suggestions for posting back to HaloPSA.

- **Nodes Involved:**  
  - Note: HTML Note (sticky note)  
  - Build AI HTML Note (Code node)

- **Node Details:**

  - **Note: HTML Note**  
    - Type: Sticky Note  
    - Role: Explains branding variables like logo URL, brand name, and output variables (`note_html`, `ticket_id`).  
    - Inputs/Outputs: None

  - **Build AI HTML Note**  
    - Type: Code (JavaScript)  
    - Role:  
      - Fetches `ticket_id` from earlier extraction node output.  
      - Uses parsed AI fields (`summary`, `next_step`, `troubleshooting`) to build styled HTML content.  
      - Includes branding elements (logo URL, brand name), colors, and footer text.  
    - Inputs: Output of Parse AI JSON and Extract Ticket (for ticket_id)  
    - Outputs: JSON with `ticket_id` and `note_html` (HTML string)  
    - Failure modes:  
      - Missing ticket_id causes error.  
      - Missing AI fields result in empty or placeholder content.

#### 2.8 Payload Wrapping

- **Overview:**  
  Wraps the HTML note and ticket metadata into the JSON structure expected by HaloPSA’s Actions API.

- **Nodes Involved:**  
  - Note: Wrap (sticky note)  
  - Wrap for Halo (Code node)

- **Node Details:**

  - **Note: Wrap**  
    - Type: Sticky Note  
    - Role: Explains purpose of wrapping payload for HaloPSA Actions API; mentions modifiable flags like visibility and outcome.  
    - Inputs/Outputs: None

  - **Wrap for Halo**  
    - Type: Code (JavaScript)  
    - Role:  
      - Constructs array with one object containing `ticket_id`, `note_html`, `is_visible_to_user: false` (private note), and `outcome: 'Private Note'`.  
    - Inputs: Output of Build AI HTML Note  
    - Outputs: JSON payload for API  
    - Failure modes:  
      - Missing fields cause malformed payload.

#### 2.9 Posting Back to HaloPSA

- **Overview:**  
  Sends the note to HaloPSA via HTTP POST to create a private note on the ticket.

- **Nodes Involved:**  
  - Note: Halo HTTP (sticky note)  
  - HaloPSA: Create Note (HTTP Request node)

- **Node Details:**

  - **Note: Halo HTTP**  
    - Type: Sticky Note  
    - Role: Instructions to set URL and authentication for HaloPSA API call.  
    - Inputs/Outputs: None

  - **HaloPSA: Create Note**  
    - Type: HTTP Request  
    - Role:  
      - Posts the wrapped note payload as JSON to HaloPSA Actions API endpoint.  
      - Uses HTTP method POST.  
      - Headers include Content-Type and Authorization Bearer token.  
    - Configuration:  
      - URL: `https://YOUR_HALO_DOMAIN/api/actions` (replace placeholder)  
      - Auth: API token in Authorization header (`Bearer YOUR_HALO_API_TOKEN`)  
      - Body: JSON from Wrap for Halo node  
    - Inputs: Output of Wrap for Halo  
    - Outputs: HTTP response (success or error)  
    - Failure modes:  
      - Authentication failure if API token invalid or missing.  
      - Network or API endpoint errors.  
      - Payload structure errors cause API rejection.

---

### 3. Summary Table

| Node Name             | Node Type                     | Functional Role               | Input Node(s)       | Output Node(s)         | Sticky Note                                                                                             |
|-----------------------|-------------------------------|------------------------------|---------------------|------------------------|-------------------------------------------------------------------------------------------------------|
| Note: Webhook         | Sticky Note                   | Webhook setup instructions   |                     | Webhook                | ### 🔔 Webhook (HaloPSA → n8n)\nUse **POST**. Replace `WEBHOOK_PATH` below, then paste the full **Production URL** back into HaloPSA webhook.\n- Path: something unique, e.g. `halopsa-new-ticket-ai`\n- HaloPSA payload must include `ticket`, `team` or `team_id` if you use the guard. |
| Webhook               | Webhook                      | Receives ticket data          | Note: Webhook       | Guard                  |                                                                                                       |
| Note: Guard           | Sticky Note                   | Guard filtering instructions | Webhook             | Guard                  | ### 🚧 Guard (optional)\nSkips tickets for a team you don’t want to process.\n- Change `teamName` or `teamId` checks below (e.g. `sales` or `6`).\n- Remove this node if you don’t need filtering. |
| Guard                 | Code                         | Filters tickets by team      | Webhook             | Extract Ticket          |                                                                                                       |
| Note: Extract         | Sticky Note                   | Ticket extraction instructions| Guard                | Extract Ticket          | ### 📦 Extract Ticket\nPulls `id`, `summary`, `details` from webhook body.\nNo keys to change unless your webhook payload uses different field names. |
| Extract Ticket        | Code                         | Extracts ticket fields       | Guard                | Build AI Prompt         |                                                                                                       |
| Note: Prompt          | Sticky Note                   | AI prompt build instructions | Extract Ticket       | Build AI Prompt         | ### 🧠 Build AI Prompt (Gemini / any LLM)\nEdit wording and the **tools/systems** your MSP uses.\nNo secrets here.\nOutputs a single `prompt` string. |
| Build AI Prompt       | Code                         | Constructs AI prompt string  | Extract Ticket       | AI Agent                |                                                                                                       |
| Note: AI Agent        | Sticky Note                   | AI agent usage instructions  | Build AI Prompt      | AI Agent                | ### 🤖 AI Agent (LangChain)\nConnect your **LLM node** here.\n- If using Gemini: add a *Google Gemini Chat Model* node and connect as the **Language Model** input.\n- Or swap for OpenAI/other LLM nodes.\n- **Set credentials in the model node, not here.** |
| AI Agent              | LangChain Agent              | Runs AI prompt on LLM        | Build AI Prompt      | Parse AI JSON           |                                                                                                       |
| Note: LLM Model       | Sticky Note                   | LLM model credential info    |                     | Google Gemini Chat Model| ### 🧩 Gemini / LLM Model Node\n**Set your API credentials** here.\n- Example: Google Gemini Chat Model\n- No other changes required. |
| Google Gemini Chat Model| LangChain Google Gemini Chat| Provides AI completion       |                     | AI Agent (languageModel)|                                                                                                       |
| Note: Parse           | Sticky Note                   | AI output parsing instructions| AI Agent             | Parse AI JSON           | ### 📑 Parse LLM Output (JSON)\nStrips ```json fences if present and parses to fields:\n- `summary`, `next_step`, `troubleshooting` (HTML), `ticket_id`. |
| Parse AI JSON         | Code                         | Parses AI JSON output        | AI Agent             | Build AI HTML Note      |                                                                                                       |
| Note: HTML Note       | Sticky Note                   | HTML note build instructions | Parse AI JSON        | Build AI HTML Note      | ### 🧱 Build HTML Note (branding)\nChange **logo URL**, colours, footer text.\nOutput: `note_html` + `ticket_id`. |
| Build AI HTML Note    | Code                         | Creates branded HTML note    | Parse AI JSON, Extract Ticket | Wrap for Halo          |                                                                                                       |
| Note: Wrap            | Sticky Note                   | Payload wrapping instructions| Build AI HTML Note   | Wrap for Halo           | ### 📦 Wrap for HaloPSA\nPrepares the **Actions API** payload for a Private Note.\n- Change `is_visible_to_user` / `outcome` if needed. |
| Wrap for Halo         | Code                         | Prepares JSON payload for API| Build AI HTML Note   | HaloPSA: Create Note    |                                                                                                       |
| Note: Halo HTTP       | Sticky Note                   | HTTP request setup instructions| Wrap for Halo       | HaloPSA: Create Note    | ### 🌐 HTTP → HaloPSA\nPOST to your HaloPSA **Actions** endpoint.\n- Base URL: `https://YOUR_HALO_DOMAIN/api/actions`\n- Auth Header: set your API token or Basic auth.\n**Replace placeholders below** in URL & Headers. |
| HaloPSA: Create Note  | HTTP Request                 | Posts note back to HaloPSA   | Wrap for Halo        |                        |                                                                                                       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: Set a unique path, e.g., `halopsa-new-ticket-ai`  
   - Purpose: Receive new ticket data from HaloPSA webhook.

2. **Add Guard Code Node (Optional Filtering)**  
   - Type: Code  
   - JavaScript logic:  
     ```js
     const body = $json.body || {};
     const teamName = String(body.team ?? body.ticket?.team ?? '').toLowerCase();
     const teamId = Number(body.team_id ?? body.ticket?.team_id ?? NaN);
     const isFiltered = teamName === 'sales' || teamId === 6; // Customize as needed
     if (isFiltered) return [];
     return $input.all();
     ```  
   - Connect Webhook output to Guard input.

3. **Add Extract Ticket Code Node**  
   - Type: Code  
   - Extract `ticket` object from input JSON.  
   - Throw error if `ticket` missing.  
   - Build `summaryCard` string with ticket ID, summary, details.  
   - Output JSON: `{ ticket_id, summary_card, details, subject }`  
   - Connect Guard output to this node.

4. **Add Build AI Prompt Code Node**  
   - Type: Code  
   - Build prompt string embedding ticket info and MSP systems, e.g.:  
     ```
     You are a senior MSP engineer. Analyze the support ticket and return **strict JSON** with: summary, next_step, troubleshooting_suggestions (HTML list), system_actions (optional HTML list).

     ### Systems we use
     1. NinjaOne (RMM)
     2. Microsoft 365 (tenant management)
     3. CIPP (multi-tenant admin)

     ### Ticket
     - ID: {ticket_id}
     - Subject: {subject}
     - Details:
     {details}
     ```  
   - Output JSON with `prompt` and `ticket_id`.  
   - Connect Extract Ticket output here.

5. **Add LangChain Agent Node**  
   - Type: LangChain Agent  
   - Text input: `{{$json.prompt}}`  
   - Prompt type: define  
   - Connect Build AI Prompt output here.

6. **Add Google Gemini Chat Model Node**  
   - Type: LangChain Google Gemini Chat Model  
   - Configure your Google API credentials here.  
   - Connect this node as the `ai_languageModel` input of the AI Agent node.

7. **Add Parse AI JSON Code Node**  
   - Type: Code  
   - Logic to strip markdown fences from AI output and parse JSON:  
     ```js
     const raw = $json.output ?? '';
     let clean = raw.trim();
     if (clean.startsWith('```')) {
       clean = clean.replace(/^```json\s*/i, '').replace(/^```\s*/i, '').replace(/```$/, '').trim();
     }
     let parsed;
     try { parsed = JSON.parse(clean); } catch (e) { throw new Error('Failed to parse LLM JSON: ' + e.message); }
     return [{ json: { summary: parsed.summary, next_step: parsed.next_step, troubleshooting: parsed.troubleshooting_suggestions, ticket_id: $json.ticket_id } }];
     ```  
   - Connect AI Agent output here.

8. **Add Build AI HTML Note Code Node**  
   - Type: Code  
   - Retrieve `ticket_id` from Extract Ticket node output.  
   - Use parsed AI fields to create branded HTML note. Customize logo URL and brand name.  
   - Output JSON: `{ ticket_id, note_html }`  
   - Example snippet:  
     ```js
     const ticket_id = $items('Extract Ticket', 0, 0)[0]?.json?.ticket_id;
     const summary = String($json.summary || '').trim();
     const next_step = String($json.next_step || '').trim();
     const troubleshooting_html = String($json.troubleshooting || '').trim();
     const LOGO_URL = 'https://YOUR_LOGO_URL/logo.png';
     const BRAND = 'Your MSP Brand';
     const note_html = `<div style="..."> ... </div>`;
     return [{ json: { ticket_id, note_html } }];
     ```  
   - Connect Parse AI JSON output here.

9. **Add Wrap for Halo Code Node**  
   - Type: Code  
   - Wrap the HTML note in a payload array for HaloPSA Actions API:  
     ```js
     return [{ json: { payload: [{ ticket_id: $json.ticket_id, note_html: $json.note_html, is_visible_to_user: false, outcome: 'Private Note' }] } }];
     ```  
   - Connect Build AI HTML Note output here.

10. **Add HTTP Request Node (HaloPSA: Create Note)**  
    - Type: HTTP Request  
    - HTTP Method: POST  
    - URL: `https://YOUR_HALO_DOMAIN/api/actions` (replace placeholder)  
    - Headers:  
      - Content-Type: application/json  
      - Authorization: Bearer YOUR_HALO_API_TOKEN (replace with valid token)  
    - Body Content: JSON (expression) `{{$json.payload}}`  
    - Connect Wrap for Halo output here.

11. **Connect all nodes as per above flow:**  
    Webhook → Guard → Extract Ticket → Build AI Prompt → AI Agent → Parse AI JSON → Build AI HTML Note → Wrap for Halo → HaloPSA: Create Note

12. **Activate the workflow and test by sending POST requests matching HaloPSA payload schema.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                             | Context or Link                                                                                      |
|------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| The webhook expects a POST payload containing at least a `ticket` object with fields `id`, `summary`, and `details`.                    | Sticky Note: Note: Webhook                                                                           |
| Customize the `teamName` or `teamId` in the Guard node to filter out tickets from specific teams you do not want to process.             | Sticky Note: Note: Guard                                                                             |
| The AI prompt includes MSP tools your company uses; edit this list to reflect your environment for better AI accuracy.                   | Sticky Note: Note: Prompt                                                                            |
| Credentials for the Gemini Chat Model must be configured in the Google Gemini Chat Model node, not in the AI Agent node.                 | Sticky Note: Note: LLM Model                                                                         |
| The HTML note includes branding elements (logo URL, brand name) that you should customize to your MSP’s branding.                        | Sticky Note: Note: HTML Note                                                                         |
| The final HTTP request node requires setting the HaloPSA API domain and an API token with appropriate permissions for creating notes.    | Sticky Note: Note: Halo HTTP                                                                         |
| [HaloPSA API Documentation](https://docs.halopsa.com/api) – Refer for API endpoint details and authentication methods.                   | External resource for HaloPSA API                                                                    |

---

**Disclaimer:** The above content is generated solely based on an n8n workflow definition. It respects all content policies and contains no illegal or protected data.