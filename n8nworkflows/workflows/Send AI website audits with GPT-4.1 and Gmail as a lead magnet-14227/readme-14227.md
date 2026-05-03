Send AI website audits with GPT-4.1 and Gmail as a lead magnet

https://n8nworkflows.xyz/workflows/send-ai-website-audits-with-gpt-4-1-and-gmail-as-a-lead-magnet-14227


# Send AI website audits with GPT-4.1 and Gmail as a lead magnet

# 1. Workflow Overview

This workflow automatically turns a website URL submitted by a lead into a personalized AI-generated website/business audit email, then sends that report via Gmail to the lead.

Its main use case is lead generation: a prospect fills in a form with their email address and website, and the workflow delivers a tailored “free audit” that demonstrates expertise and creates a natural path toward booking a sales call.

The workflow is organized into six logical blocks:

## 1.1 Lead Capture
The workflow starts with an n8n form that collects the prospect’s email address and website URL.

## 1.2 Website Retrieval and Text Extraction
The submitted website is fetched via HTTP, then cleaned into plain text so the AI model can analyze readable content instead of raw HTML.

## 1.3 AI Diagnostic Generation
A first AI agent analyzes the extracted website text and returns a structured JSON diagnostic describing business type, offers, audience, channels, funnels, pain points, and copywriting assets.

## 1.4 AI Email Writing
A second AI agent takes the structured diagnostic and produces a commercial outreach email in the same language as the prospect’s website.

## 1.5 HTML Email Branding and Rendering
The generated message is converted into a branded HTML email template with visual formatting, CTA button, logo, and footer links.

## 1.6 Email Delivery
The final HTML email is sent through Gmail to the address provided in the form.

---

# 2. Block-by-Block Analysis

## 2.1 Lead Capture

### Overview
This block exposes the public entry point of the workflow. It collects the recipient’s email address and target website URL through an n8n-hosted form.

### Nodes Involved
- Form Trigger

### Node Details

#### Form Trigger
- **Type and technical role:** `n8n-nodes-base.formTrigger`  
  Public workflow trigger that starts execution when the form is submitted.
- **Configuration choices:**
  - Form title: **Free AI Business Audit**
  - Two form fields:
    - `email` as an email-type field, required
    - `website` as a text field, required, with placeholder showing a URL format
- **Key expressions or variables used:**
  - Downstream nodes reference:
    - `$('Form Trigger').item.json.email`
    - `$('Form Trigger').item.json.website`
- **Input and output connections:**
  - Input: none, entry point
  - Output: connected to **Fetch Website HTML**
- **Version-specific requirements:**
  - Uses **typeVersion 2.2**
  - Requires an n8n version that supports the Form Trigger node and hosted form/webhook behavior
- **Edge cases or potential failure types:**
  - User submits malformed or unreachable URL
  - Email field is syntactically valid but operationally undeliverable
  - Public form can attract spam if no filtering/rate limiting is added
- **Sub-workflow reference:** none

---

## 2.2 Website Retrieval and Text Extraction

### Overview
This block retrieves the HTML of the submitted website and transforms it into plain text plus lightweight behavioral signals. It prepares a compact, analysis-friendly payload for the analyst agent.

### Nodes Involved
- Fetch Website HTML
- Clean HTML

### Node Details

#### Fetch Website HTML
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Fetches the raw HTML content of the submitted website with an HTTP GET request.
- **Configuration choices:**
  - URL is dynamically set from the form submission:
    - `={{ $('Form Trigger').item.json.website }}`
  - No custom headers, auth, redirect handling, timeout tuning, or retry strategy are explicitly configured
- **Key expressions or variables used:**
  - Input URL from `Form Trigger`
- **Input and output connections:**
  - Input: **Form Trigger**
  - Output: **Clean HTML**
- **Version-specific requirements:**
  - Uses **typeVersion 4.2**
  - Behavior may vary depending on n8n’s HTTP Request implementation in the installed version
- **Edge cases or potential failure types:**
  - Invalid URL format
  - DNS/network failure
  - SSL/TLS certificate issues
  - 403/404/5xx responses
  - Websites that block bots or require JavaScript rendering
  - Redirect loops or anti-bot pages
  - Large pages causing slow responses
- **Sub-workflow reference:** none

#### Clean HTML
- **Type and technical role:** `n8n-nodes-base.code`  
  JavaScript transformation node that strips HTML, scripts, styles, and comments; limits content length; and extracts useful website signals.
- **Configuration choices:**
  - Reads HTML from:
    - `$input.first().json.data || ''`
  - Removes:
    - `<script>` blocks
    - `<style>` blocks
    - HTML comments
    - all HTML tags
  - Normalizes whitespace
  - Truncates cleaned text to **28,000 characters**
  - Builds a `signals` object with:
    - `url`
    - `page_title`
    - `has_wa`
    - `has_tel`
    - `has_mailto`
    - `has_checkout`
    - `has_booking`
    - `social`
- **Key expressions or variables used:**
  - `$input.first().json.data`
  - `$json.url`
  - `$json.pageTitle`
  - Regex-based checks against original HTML
- **Input and output connections:**
  - Input: **Fetch Website HTML**
  - Output: **Analyst Agent**
- **Version-specific requirements:**
  - Uses **typeVersion 2**
  - Requires Code node JavaScript execution support
- **Edge cases or potential failure types:**
  - Some HTTP responses may not populate `json.data` as expected depending on response format
  - Dynamic sites with little server-rendered HTML may produce nearly empty text
  - Truncation may remove important context on large sites
  - Regex signal detection may miss uncommon implementations
  - `pageTitle` and `url` may be absent from upstream response data
- **Sub-workflow reference:** none

---

## 2.3 AI Diagnostic Generation

### Overview
This block uses a language model-powered agent to analyze the cleaned site content and produce a structured JSON diagnostic. The result is constrained by a structured output parser so downstream automation can rely on stable fields.

### Nodes Involved
- Analyst Agent
- OpenAI – Analyst Model
- Analyst Output Parser

### Node Details

#### Analyst Agent
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent`  
  Central AI reasoning node that receives cleaned site content and instructs the model to generate a strict diagnostic JSON.
- **Configuration choices:**
  - Prompt type: defined prompt
  - User prompt includes:
    - `url`
    - `page_title`
    - `text`
    - serialized `signals`
  - Has output parser enabled
  - Extensive system message defines:
    - target business categories
    - heuristics for contact channel detection
    - pain point generation rules
    - confidence calibration
    - strict output schema
    - language/readability inference
- **Key expressions or variables used:**
  - `{{ $json.url }}`
  - `{{ $json.pageTitle }}`
  - `{{ $json.text }}`
  - `{{ JSON.stringify($json.signals) }}`
- **Input and output connections:**
  - Main input: **Clean HTML**
  - AI language model input: **OpenAI – Analyst Model**
  - AI output parser input: **Analyst Output Parser**
  - Main output: **Writer Agent**
- **Version-specific requirements:**
  - Uses **typeVersion 2.2**
  - Requires n8n LangChain-compatible AI nodes
  - Depends on agent node support for linked model and structured parser
- **Edge cases or potential failure types:**
  - Model may still return invalid JSON if prompt adherence fails
  - Thin or noisy source content may reduce diagnostic quality
  - Token limits could be approached if upstream text becomes large
  - Hallucinated inferences remain possible despite instructions
  - If the schema and actual output drift, parsing can fail
- **Sub-workflow reference:** none

#### OpenAI – Analyst Model
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
  Supplies the chat model used by the analyst agent.
- **Configuration choices:**
  - Model: **gpt-5.2**
  - No advanced options explicitly set
  - Uses OpenAI credentials named **Synecta SJ**
- **Key expressions or variables used:** none
- **Input and output connections:**
  - Output via AI language model connection to **Analyst Agent**
- **Version-specific requirements:**
  - Uses **typeVersion 1.2**
  - Requires valid OpenAI API credentials and model availability in the account
- **Edge cases or potential failure types:**
  - Authentication failure
  - Model unavailable for the API project
  - Rate limiting or quota exhaustion
  - OpenAI API timeouts/errors
- **Sub-workflow reference:** none

#### Analyst Output Parser
- **Type and technical role:** `@n8n/n8n-nodes-langchain.outputParserStructured`  
  Validates and structures the analyst model output according to a JSON schema example.
- **Configuration choices:**
  - Schema includes two top-level objects:
    - `diagnostic`
    - `copy_kit`
  - Enforces nested structures such as:
    - `pain_points`
    - `cta_inventory`
    - `audience_jtbd`
    - `improvement_candidates`
- **Key expressions or variables used:** none
- **Input and output connections:**
  - Output parser connection to **Analyst Agent**
- **Version-specific requirements:**
  - Uses **typeVersion 1.3**
  - Requires parser compatibility with agent node structured outputs
- **Edge cases or potential failure types:**
  - Parsing failure if model output is malformed or incomplete
  - Field type mismatch
  - Unexpected extra wrappers or markdown from the model
- **Sub-workflow reference:** none

---

## 2.4 AI Email Writing

### Overview
This block converts the structured business diagnostic into a personalized sales email. It uses the detected website language to match the lead’s language automatically.

### Nodes Involved
- Writer Agent
- OpenAI – Writer Model
- Writer Output Parser

### Node Details

#### Writer Agent
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent`  
  Takes the analyst output and generates a strict JSON object containing subject, preheader, and HTML email body.
- **Configuration choices:**
  - Prompt includes serialized analyst output:
    - `{{ JSON.stringify($json.output) }}`
  - Optional parameters are passed inline:
    - channel: email
    - length: standard
    - style: direct
    - language from `output.diagnostic.meta.lang_detected`, default `en`
  - System message enforces:
    - compliment-based introduction
    - exactly 10 improvements
    - same-language output as website
    - concise persuasive tone
    - strict JSON output with `subject`, `preheader`, `message_html`
    - plain HTML without styles/buttons/signature
- **Key expressions or variables used:**
  - `{{ JSON.stringify($json.output) }}`
  - `{{ $json.output.diagnostic.meta.lang_detected || 'en' }}`
- **Input and output connections:**
  - Main input: **Analyst Agent**
  - AI language model input: **OpenAI – Writer Model**
  - AI output parser input: **Writer Output Parser**
  - Main output: **Build Email HTML**
- **Version-specific requirements:**
  - Uses **typeVersion 2.2**
  - Requires LangChain agent support in n8n
- **Edge cases or potential failure types:**
  - If analyst output is incomplete, the writer may struggle to generate 10 strong improvements
  - Invalid JSON output may fail structured parsing
  - Language detection may be wrong on multilingual sites
  - Writer may produce HTML structure that does not fully match downstream assumptions
- **Sub-workflow reference:** none

#### OpenAI – Writer Model
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
  Supplies the language model used by the writer agent.
- **Configuration choices:**
  - Model: **gpt-4.1-mini**
  - No additional model options configured
  - Uses OpenAI credentials named **Synecta SJ**
- **Key expressions or variables used:** none
- **Input and output connections:**
  - Output via AI language model connection to **Writer Agent**
- **Version-specific requirements:**
  - Uses **typeVersion 1.2**
  - Requires access to the selected OpenAI model
- **Edge cases or potential failure types:**
  - Authentication or quota errors
  - Model deprecation/unavailability
  - API latency or timeouts
- **Sub-workflow reference:** none

#### Writer Output Parser
- **Type and technical role:** `@n8n/n8n-nodes-langchain.outputParserStructured`  
  Forces the writer model output into a predictable JSON object for later rendering.
- **Configuration choices:**
  - Schema includes:
    - `subject`
    - `preheader`
    - `message_html`
- **Key expressions or variables used:** none
- **Input and output connections:**
  - Output parser connection to **Writer Agent**
- **Version-specific requirements:**
  - Uses **typeVersion 1.3**
- **Edge cases or potential failure types:**
  - Fails if the model emits invalid HTML strings or malformed JSON
  - Fails if required fields are missing
- **Sub-workflow reference:** none

---

## 2.5 HTML Email Branding and Rendering

### Overview
This block transforms the writer’s plain HTML content into a polished branded email template. It wraps the content in a full HTML document, converts the ordered list into styled cards, and appends a CTA and branded footer.

### Nodes Involved
- Build Email HTML

### Node Details

#### Build Email HTML
- **Type and technical role:** `n8n-nodes-base.code`  
  JavaScript node that post-processes generated email content into branded, email-safe HTML.
- **Configuration choices:**
  - Editable constants at the top:
    - `YOUR_NAME`
    - `YOUR_EMAIL`
    - `CAL_URL`
    - `INSTAGRAM_URL`
    - `LINKEDIN_URL`
    - `LOGO_URL`
  - Reads source HTML from:
    - `($input.first().json.output && $input.first().json.output.message_html) || ''`
  - Includes brand colors and font constants
  - Detects prior processing with marker:
    - `<!--SYNECTA-CARDS-->`
  - Removes inline styles from links and reapplies standardized link styles
  - Converts `<ol><li>` elements into card-based table layouts
  - Injects:
    - branded header with logo
    - main content area
    - CTA button linking to booking calendar
    - footer with social icons and support text
  - Returns:
    - `{ email_html: html }`
- **Key expressions or variables used:**
  - `$input.first().json.output.message_html`
  - Hard-coded branding constants
- **Input and output connections:**
  - Input: **Writer Agent**
  - Output: **Send Email**
- **Version-specific requirements:**
  - Uses **typeVersion 2**
  - Requires Code node support
- **Edge cases or potential failure types:**
  - If writer output lacks `<ol><li>` structure, numbered cards will not render as intended
  - If `LOGO_URL` or social icon/image URLs fail, visuals break but email may still send
  - Some email clients may partially ignore modern CSS features like gradients or shadows
  - Hard-coded `lang="en"` in the final HTML does not reflect actual message language
  - CTA text is always English: **Book a Free Call**
- **Sub-workflow reference:** none

---

## 2.6 Email Delivery

### Overview
This final block sends the rendered HTML email to the submitted email address using Gmail OAuth2 credentials.

### Nodes Involved
- Send Email

### Node Details

#### Send Email
- **Type and technical role:** `n8n-nodes-base.gmail`  
  Sends the completed branded HTML email through a connected Gmail account.
- **Configuration choices:**
  - Recipient:
    - `={{ $('Form Trigger').item.json.email }}`
  - Message body:
    - `={{ $json.email_html }}`
  - Subject:
    - `=🕵🏻‍♂️ {{ $('Writer Agent').item.json.output.subject }}`
  - Options:
    - sender name: **Stefan Joulien – Synecta**
    - append attribution: disabled
  - Uses Gmail OAuth2 credential named **stefan@synecta.ai**
- **Key expressions or variables used:**
  - `$('Form Trigger').item.json.email`
  - `$json.email_html`
  - `$('Writer Agent').item.json.output.subject`
- **Input and output connections:**
  - Input: **Build Email HTML**
  - Output: none
- **Version-specific requirements:**
  - Uses **typeVersion 2.1**
  - Requires Gmail OAuth2 credentials with send permissions
- **Edge cases or potential failure types:**
  - OAuth token expiration or revoked access
  - Gmail daily sending quotas
  - HTML emails landing in spam/promotions
  - Invalid destination email
  - Subject missing if upstream writer output fails
- **Sub-workflow reference:** none

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Form Trigger | n8n-nodes-base.formTrigger | Public form entry point collecting email and website URL |  | Fetch Website HTML | ## 📝 Step 1 – Form Trigger<br>A simple n8n form with two fields:<br>- **Email** → where the report is sent<br>- **Website URL** → the site to be analysed<br><br>💡 You can embed this form on your landing page or share the direct URL. |
| Fetch Website HTML | n8n-nodes-base.httpRequest | Downloads raw HTML from the submitted website | Form Trigger | Clean HTML | ## 🌐 Step 2 – Fetch & Clean Website<br>**Fetch Website HTML:** Makes an HTTP GET request to the submitted URL and retrieves the raw HTML.<br><br>**Clean HTML:** Strips all tags, scripts and styles, leaving only readable plain text (max 28,000 chars ≈ 7k tokens) + key signals (WhatsApp, booking links, checkout, social). |
| Clean HTML | n8n-nodes-base.code | Converts website HTML into plain text and extracts site signals | Fetch Website HTML | Analyst Agent | ## 🌐 Step 2 – Fetch & Clean Website<br>**Fetch Website HTML:** Makes an HTTP GET request to the submitted URL and retrieves the raw HTML.<br><br>**Clean HTML:** Strips all tags, scripts and styles, leaving only readable plain text (max 28,000 chars ≈ 7k tokens) + key signals (WhatsApp, booking links, checkout, social). |
| OpenAI – Analyst Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | Provides the LLM used by the analyst agent |  | Analyst Agent | ## 🔍 Step 3 – Analyst Agent<br>Reads the plain-text website content and returns a **structured JSON diagnostic** with:<br>- Business type, offers & target audience<br>- Contact channels & conversion funnels<br>- 7–10 automatable pain points with evidence & likelihood scores<br>- Copy kit: compliments, hook angles, tone, objections & improvement candidates<br><br>⚙️ Uses **gpt-4o-mini** by default. Swap for gpt-4o for higher accuracy. |
| Analyst Output Parser | @n8n/n8n-nodes-langchain.outputParserStructured | Enforces structured JSON output for the analyst agent |  | Analyst Agent | ## 🔍 Step 3 – Analyst Agent<br>Reads the plain-text website content and returns a **structured JSON diagnostic** with:<br>- Business type, offers & target audience<br>- Contact channels & conversion funnels<br>- 7–10 automatable pain points with evidence & likelihood scores<br>- Copy kit: compliments, hook angles, tone, objections & improvement candidates<br><br>⚙️ Uses **gpt-4o-mini** by default. Swap for gpt-4o for higher accuracy. |
| Analyst Agent | @n8n/n8n-nodes-langchain.agent | Produces structured business/site diagnostic and copy kit | Clean HTML | Writer Agent | ## 🔍 Step 3 – Analyst Agent<br>Reads the plain-text website content and returns a **structured JSON diagnostic** with:<br>- Business type, offers & target audience<br>- Contact channels & conversion funnels<br>- 7–10 automatable pain points with evidence & likelihood scores<br>- Copy kit: compliments, hook angles, tone, objections & improvement candidates<br><br>⚙️ Uses **gpt-4o-mini** by default. Swap for gpt-4o for higher accuracy. |
| OpenAI – Writer Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | Provides the LLM used by the writer agent |  | Writer Agent | ## ✍️ Step 4 – Writer Agent<br>Takes the analyst's JSON and writes a **personalised email** with:<br>- A genuine opening compliment based on the site analysis<br>- 10 numbered improvements in prose (benefit-focused, concise)<br>- A closing CTA to book a discovery call<br><br>⚠️ The email is automatically written in the **same language as the lead's website** (e.g. Spanish site → Spanish email). |
| Writer Output Parser | @n8n/n8n-nodes-langchain.outputParserStructured | Enforces structured JSON output for the writer agent |  | Writer Agent | ## ✍️ Step 4 – Writer Agent<br>Takes the analyst's JSON and writes a **personalised email** with:<br>- A genuine opening compliment based on the site analysis<br>- 10 numbered improvements in prose (benefit-focused, concise)<br>- A closing CTA to book a discovery call<br><br>⚠️ The email is automatically written in the **same language as the lead's website** (e.g. Spanish site → Spanish email). |
| Writer Agent | @n8n/n8n-nodes-langchain.agent | Writes the outreach email from the diagnostic JSON | Analyst Agent | Build Email HTML | ## ✍️ Step 4 – Writer Agent<br>Takes the analyst's JSON and writes a **personalised email** with:<br>- A genuine opening compliment based on the site analysis<br>- 10 numbered improvements in prose (benefit-focused, concise)<br>- A closing CTA to book a discovery call<br><br>⚠️ The email is automatically written in the **same language as the lead's website** (e.g. Spanish site → Spanish email). |
| Build Email HTML | n8n-nodes-base.code | Wraps the generated message in a branded HTML email template | Writer Agent | Send Email | ## 🎨 Step 5 – Build Email HTML<br>Converts the writer's plain message into a **branded HTML email** with:<br>- Gradient header with your logo<br>- Numbered improvement cards with brand colours<br>- CTA button linking to your booking calendar<br>- Dark footer with email, Instagram & LinkedIn icons<br><br>⚙️ **Configure your details at the top of this node:**<br>`YOUR_NAME, YOUR_EMAIL, CAL_URL,`<br>`INSTAGRAM_URL, LINKEDIN_URL, LOGO_URL` |
| Send Email | n8n-nodes-base.gmail | Sends the final HTML audit email through Gmail | Build Email HTML |  | ## 📧 Step 6 – Send Email<br>Sends the branded HTML report via **Gmail** to the address submitted in the form.<br><br>🔑 **Setup required:**<br>1. Connect your Gmail account under Credentials<br>2. Update `senderName` to your name in the node settings<br>3. The email subject is dynamically generated by the Writer Agent |
| Sticky – Overview | n8n-nodes-base.stickyNote | Visual documentation block summarizing the workflow purpose |  |  | ## 🤖 AI Business Analysis – Lead Magnet<br>**What this workflow does:**<br>A prospect fills in a form with their email and website URL. The workflow fetches their website, analyses it with AI, writes a personalised 10-improvement email in their language, and sends it automatically via Gmail.<br><br>**Estimated run time:** ~30–60 seconds per lead. |
| Sticky – Form | n8n-nodes-base.stickyNote | Visual documentation for the form block |  |  |  |
| Sticky – Fetch & Clean | n8n-nodes-base.stickyNote | Visual documentation for fetch/clean block |  |  |  |
| Sticky – Analyst Agent | n8n-nodes-base.stickyNote | Visual documentation for analyst AI block |  |  |  |
| Sticky – Writer Agent | n8n-nodes-base.stickyNote | Visual documentation for writer AI block |  |  |  |
| Sticky – Build HTML | n8n-nodes-base.stickyNote | Visual documentation for HTML rendering block |  |  |  |
| Sticky – Send Email | n8n-nodes-base.stickyNote | Visual documentation for email sending block |  |  |  |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it something like: **AI Business Analysis Lead Magnet – Auto Email Report**.
   - Ensure AI nodes are available in your n8n instance.

2. **Add a Form Trigger node**
   - Node type: **Form Trigger**
   - Set the form title to: **Free AI Business Audit**
   - Add two form fields:
     1. Email field
        - Type: email
        - Label: `email`
        - Placeholder: `your@email.com`
        - Required: enabled
     2. Website field
        - Type: text
        - Label: `website`
        - Placeholder: `https://yourwebsite.com`
        - Required: enabled

3. **Add an HTTP Request node**
   - Name: **Fetch Website HTML**
   - Node type: **HTTP Request**
   - Set URL to expression:
     - `{{ $('Form Trigger').item.json.website }}`
   - Leave it as a GET request with default settings unless you need custom headers or timeout handling.
   - Connect:
     - **Form Trigger → Fetch Website HTML**

4. **Add a Code node for HTML cleaning**
   - Name: **Clean HTML**
   - Node type: **Code**
   - Paste JavaScript that:
     - reads the HTML response
     - removes scripts, styles, comments, and tags
     - normalizes whitespace
     - limits text to 28,000 characters
     - returns:
       - `url`
       - `pageTitle`
       - `text`
       - `signals`
   - Use this logic:
     - source HTML from `$input.first().json.data || ''`
     - signal detection for WhatsApp, tel, mailto, checkout, booking, and social links
   - Connect:
     - **Fetch Website HTML → Clean HTML**

5. **Add the analyst language model node**
   - Name: **OpenAI – Analyst Model**
   - Node type: **OpenAI Chat Model** from LangChain
   - Select your OpenAI credentials
   - Choose model: **gpt-5.2**
   - Leave advanced options empty unless you want to control temperature or other inference settings

6. **Add the analyst structured output parser**
   - Name: **Analyst Output Parser**
   - Node type: **Structured Output Parser**
   - Define a JSON schema example containing:
     - `diagnostic.business_type`
     - `diagnostic.offers`
     - `diagnostic.audience`
     - `diagnostic.contact_channels`
     - `diagnostic.funnels`
     - `diagnostic.pain_points[]`
     - `diagnostic.meta`
     - `copy_kit.compliments`
     - `copy_kit.hook_angles`
     - `copy_kit.tone_mirror`
     - `copy_kit.objections`
     - `copy_kit.cta_inventory`
     - `copy_kit.audience_jtbd`
     - `copy_kit.improvement_candidates`
   - Keep the structure strict and complete.

7. **Add the analyst agent**
   - Name: **Analyst Agent**
   - Node type: **AI Agent**
   - Set prompt mode to define your own text
   - In the main prompt, pass the cleaned website data:
     - URL
     - page title
     - text
     - JSON-stringified signals
   - Enable structured output parser
   - Connect:
     - **Clean HTML → Analyst Agent**
     - **OpenAI – Analyst Model → Analyst Agent** via AI language model connection
     - **Analyst Output Parser → Analyst Agent** via AI output parser connection
   - Add a detailed system instruction that tells the model to:
     - analyze the business
     - infer business type
     - identify offers, audience, channels, funnels
     - generate 7–10 pain points with evidence and confidence
     - produce `copy_kit` content
     - output strict JSON only

8. **Add the writer language model node**
   - Name: **OpenAI – Writer Model**
   - Node type: **OpenAI Chat Model** from LangChain
   - Select the same or another OpenAI credential
   - Choose model: **gpt-4.1-mini**

9. **Add the writer structured output parser**
   - Name: **Writer Output Parser**
   - Node type: **Structured Output Parser**
   - Schema should include:
     - `subject`
     - `preheader`
     - `message_html`

10. **Add the writer agent**
    - Name: **Writer Agent**
    - Node type: **AI Agent**
    - Use the analyst agent output as input
    - Main prompt should pass:
      - `{{ JSON.stringify($json.output) }}`
      - optional parameters for channel, length, style, and language
    - Set language expression to:
      - `{{ $json.output.diagnostic.meta.lang_detected || 'en' }}`
    - Enable output parser
    - Connect:
      - **Analyst Agent → Writer Agent**
      - **OpenAI – Writer Model → Writer Agent** via AI language model connection
      - **Writer Output Parser → Writer Agent** via AI output parser connection
    - In the system instructions, require:
      - same-language email generation
      - exactly 10 numbered improvements
      - concise persuasive style
      - strict JSON output only
      - `message_html` with `<ol>` and `<li>` structure

11. **Add a Code node for branded email rendering**
    - Name: **Build Email HTML**
    - Node type: **Code**
    - Paste JavaScript that:
      - reads `output.message_html`
      - defines brand constants and links
      - transforms ordered list items into branded numbered cards
      - wraps everything in a full HTML email layout
      - adds CTA button and footer
      - returns `{ email_html: html }`
    - At the top of the code, customize:
      - `YOUR_NAME`
      - `YOUR_EMAIL`
      - `CAL_URL`
      - `INSTAGRAM_URL`
      - `LINKEDIN_URL`
      - `LOGO_URL`
    - Connect:
      - **Writer Agent → Build Email HTML**

12. **Add a Gmail node**
    - Name: **Send Email**
    - Node type: **Gmail**
    - Operation: send email
    - Configure Gmail OAuth2 credentials
    - Set recipient expression:
      - `{{ $('Form Trigger').item.json.email }}`
    - Set subject expression:
      - `🕵🏻‍♂️ {{ $('Writer Agent').item.json.output.subject }}`
    - Set message/body expression:
      - `{{ $json.email_html }}`
    - In options:
      - set sender name, for example: **Stefan Joulien – Synecta**
      - disable attribution/appended signature if desired
    - Connect:
      - **Build Email HTML → Send Email**

13. **Verify connection order**
    - Main flow:
      - **Form Trigger → Fetch Website HTML → Clean HTML → Analyst Agent → Writer Agent → Build Email HTML → Send Email**
    - AI side connections:
      - **OpenAI – Analyst Model → Analyst Agent**
      - **Analyst Output Parser → Analyst Agent**
      - **OpenAI – Writer Model → Writer Agent**
      - **Writer Output Parser → Writer Agent**

14. **Configure credentials**
    - **OpenAI credential**
      - Create or select an OpenAI API credential
      - Ensure the selected account has access to:
        - `gpt-5.2`
        - `gpt-4.1-mini`
    - **Gmail OAuth2 credential**
      - Connect the Gmail account that will send the lead magnet email
      - Ensure it has permission to send mail
    - If using production deployment, verify sender identity and domain reputation

15. **Add optional visual documentation**
    - Add sticky notes for:
      - workflow overview
      - form section
      - fetch/clean section
      - analyst AI section
      - writer AI section
      - email rendering section
      - email sending section

16. **Test with a real website**
    - Submit the form with:
      - a valid email you control
      - a reachable business website
    - Check:
      - HTTP fetch succeeds
      - Clean HTML returns readable text
      - Analyst Agent output is valid structured JSON
      - Writer Agent returns valid subject/preheader/message_html
      - Build Email HTML produces full HTML
      - Gmail sends the message successfully

17. **Harden for production**
    - Add error handling branches if needed
    - Consider validating/normalizing website URLs before fetch
    - Add retry or timeout settings for the HTTP Request node
    - Add spam protection or CAPTCHA on the form if public
    - Optionally store submissions and generated audits in Sheets, Airtable, or a database

### Sub-workflow setup
This workflow does **not** call any sub-workflow and does not contain any Execute Workflow node. There are no child workflow dependencies to configure.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| AI Business Analysis – Lead Magnet: A prospect fills in a form with their email and website URL. The workflow fetches their website, analyses it with AI, writes a personalised 10-improvement email in their language, and sends it automatically via Gmail. Estimated run time: ~30–60 seconds per lead. | Workflow overview sticky note |
| The Build Email HTML node contains hard-coded branding and booking constants that must be customized before use. | Internal branding configuration in Code node |
| The writer agent is designed to mirror the language detected from the lead’s website, but the final branded HTML wrapper still hard-codes `lang="en"` and an English CTA label. | Important implementation note |
| The analyst sticky note says the workflow uses **gpt-4o-mini** by default, but the actual configured analyst model in this JSON is **gpt-5.2**. | Configuration mismatch to be aware of |
| OpenAI credential name found in the workflow: **Synecta SJ** | Credential reference |
| Gmail credential name found in the workflow: **stefan@synecta.ai** | Credential reference |