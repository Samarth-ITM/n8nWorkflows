Generate AI sales battle cards with Olostep, Gemini, and Google Docs

https://n8nworkflows.xyz/workflows/generate-ai-sales-battle-cards-with-olostep--gemini--and-google-docs-14661


# Generate AI sales battle cards with Olostep, Gemini, and Google Docs

# 1. Workflow Overview

This workflow generates a personalized **sales battle card** for a prospect company. It starts from a form submission, performs external company research with Olostep, analyzes the results with Gemini, produces outreach assets, formats the final output as a Google Doc, and shares that document by email via Google Drive sharing.

It is designed for use cases such as:
- SDR and AE prospect preparation
- founder-led outbound
- agency outbound research
- scalable account-based personalization
- AI-assisted pre-call research and messaging

## 1.1 Input Reception
The workflow begins with a form where the user provides:
- company name
- company website URL
- the seller’s value proposition

This user input becomes the basis for all downstream research and AI prompting.

## 1.2 External Research Collection
The workflow gathers prospect data in two ways:
- Olostep searches for recent company news, hiring, and challenges
- Olostep scrapes the company website content

These two research streams provide contextual material for AI analysis.

## 1.3 AI Research Analysis
Gemini analyzes the company research against the submitted value proposition and identifies:
- business triggers
- why outreach is relevant now
- how the seller’s offer matches prospect needs

## 1.4 AI Pitch Asset Generation
A second Gemini step transforms the research analysis and website content into practical outreach assets:
- cold outreach hook
- discovery questions
- likely objection and rebuttal

## 1.5 Final Battle Card Composition
A third Gemini step combines the earlier analysis and pitch assets into a single markdown-formatted sales battle card.

## 1.6 Markdown-to-Google-Doc Conversion
The markdown output is converted to HTML, wrapped as a binary file, uploaded to Google Drive, then copied into native Google Docs format.

## 1.7 Cleanup, Sharing, and Notification
The temporary HTML file is deleted, and the resulting Google Doc is shared with a configured email recipient. The share action includes an email message with a link to the generated file.

---

# 2. Block-by-Block Analysis

## 2.1 Input Reception

**Overview:**  
This block captures the user’s inputs through an n8n form trigger. It defines the three required business inputs that drive the rest of the workflow.

**Nodes Involved:**  
- On form submission

### Node Details

#### On form submission
- **Type and technical role:** `n8n-nodes-base.formTrigger`  
  Entry-point node that exposes a hosted form and starts the workflow upon submission.
- **Configuration choices:**  
  - Form title: `Sales Battle Card for a prospect`
  - Description: `Please enter the company information`
  - Fields:
    - `Company Name`
    - `Company website URL`
    - `Value proposition` as a textarea
- **Key expressions or variables used:**  
  This node does not use inbound expressions; downstream nodes reference:
  - `$('On form submission').item.json['Company Name']`
  - `$('On form submission').item.json['Company website URL']`
  - `$('On form submission').item.json['Value proposition']`
- **Input and output connections:**  
  - Input: none
  - Output: `Company research`
- **Version-specific requirements:**  
  Uses `typeVersion: 2.5`, so form behavior depends on newer n8n form trigger capabilities.
- **Edge cases or potential failure types:**  
  - Missing or malformed website URL may break scraping
  - Empty value proposition reduces AI output quality
  - Unvalidated user input may generate weak or irrelevant prompts
- **Sub-workflow reference:**  
  None

---

## 2.2 External Research Collection

**Overview:**  
This block gathers factual and contextual prospect information. First, Olostep searches for company-related news and signals; then it scrapes the prospect website for messaging and positioning.

**Nodes Involved:**  
- Company research
- Scrape company website

### Node Details

#### Company research
- **Type and technical role:** `n8n-nodes-olostep.olostepScrape`  
  External research node using Olostep to answer a natural-language research prompt.
- **Configuration choices:**  
  - Resource mode: `answer`
  - Task prompt dynamically built from form input:
    - latest company news
    - company challenges
    - company hiring
- **Key expressions or variables used:**  
  - `{{ $json['Company Name'] }}`
  Full task:
  - `Find me the newst {{ $json['Company Name'] }} news, {{ $json['Company Name'] }} challenges, {{ $json['Company Name'] }} hiring.`
- **Input and output connections:**  
  - Input: `On form submission`
  - Output: `Scrape company website`
- **Version-specific requirements:**  
  `typeVersion: 1`
- **Edge cases or potential failure types:**  
  - Typo in prompt text: `newst` instead of `newest`; likely still understandable, but not ideal
  - Olostep auth failure
  - No useful results for obscure companies
  - Rate limiting or external API downtime
  - Output schema may vary depending on Olostep response quality
- **Sub-workflow reference:**  
  None

#### Scrape company website
- **Type and technical role:** `n8n-nodes-olostep.olostepScrape`  
  Website scraping step to capture prospect messaging from the submitted URL.
- **Configuration choices:**  
  - `url_to_scrape` is taken directly from the form submission
- **Key expressions or variables used:**  
  - `{{ $('On form submission').item.json['Company website URL'] }}`
- **Input and output connections:**  
  - Input: `Company research`
  - Output: `Research Analyzer`
- **Version-specific requirements:**  
  `typeVersion: 1`
- **Edge cases or potential failure types:**  
  - Invalid URL format
  - Website blocks scraping
  - Redirect loops, bot protection, or JavaScript-heavy content
  - Empty or partial `markdown_content`
  - The node is operationally dependent on the previous node because of the linear connection, even though data is sourced from the form
- **Sub-workflow reference:**  
  None

---

## 2.3 AI Research Analysis

**Overview:**  
This block uses Gemini to compare research findings with the seller’s value proposition. It identifies timely triggers and frames how the solution addresses them.

**Nodes Involved:**  
- Research Analyzer

### Node Details

#### Research Analyzer
- **Type and technical role:** `@n8n/n8n-nodes-langchain.googleGemini`  
  LLM processing node using Gemini 2.5 Flash for structured research analysis.
- **Configuration choices:**  
  - Model: `models/gemini-2.5-flash`
  - JSON output enabled: `true`
  - Prompt structure uses two messages:
    - user/content message containing research data and value proposition
    - model-role instruction message defining the analysis task
- **Key expressions or variables used:**  
  - `{{ $('Company research').item.json.result.result }}`
  - `{{ $('On form submission').item.json['Value proposition'] }}`
- **Input and output connections:**  
  - Input: `Scrape company website`
  - Output: `Pitch Assets generator`
- **Version-specific requirements:**  
  `typeVersion: 1.1`  
  Requires Google PaLM/Gemini credential configured in n8n’s LangChain-compatible Gemini node.
- **Edge cases or potential failure types:**  
  - Auth or quota issues with Gemini
  - Model output may not perfectly match expected structure even with JSON output enabled
  - If `result.result` is missing from Olostep output, expressions may resolve to undefined
  - Prompt quality depends heavily on research completeness
- **Sub-workflow reference:**  
  None

---

## 2.4 AI Pitch Asset Generation

**Overview:**  
This block turns the strategic analysis into practical sales messaging. It produces outreach copy, discovery prompts, and objection handling tailored to the researched company.

**Nodes Involved:**  
- Pitch Assets generator

### Node Details

#### Pitch Assets generator
- **Type and technical role:** `@n8n/n8n-nodes-langchain.googleGemini`  
  LLM generation node creating sales-ready pitch assets from prior analysis.
- **Configuration choices:**  
  - Model: `models/gemini-2.5-flash`
  - Uses prior Gemini output plus website markdown content
  - Instructs Gemini to return clean Markdown only
- **Key expressions or variables used:**  
  - `{{ $json.content.parts.last().text }}`  
    Refers to the previous node’s current item, i.e. `Research Analyzer` output
  - `{{ $('Scrape company website').item.json.markdown_content }}`
- **Input and output connections:**  
  - Input: `Research Analyzer`
  - Output: `Sales battle card generator`
- **Version-specific requirements:**  
  `typeVersion: 1.1`
- **Edge cases or potential failure types:**  
  - If Gemini response structure changes, `.content.parts.last().text` may fail
  - Empty `markdown_content` weakens the opener and talk tracks
  - Hallucinated objections or hooks if research quality is poor
- **Sub-workflow reference:**  
  None

---

## 2.5 Final Battle Card Composition

**Overview:**  
This block consolidates the earlier analysis into a single professional one-page battle card in markdown. It also injects the research sources for traceability.

**Nodes Involved:**  
- Sales battle card generator

### Node Details

#### Sales battle card generator
- **Type and technical role:** `@n8n/n8n-nodes-langchain.googleGemini`  
  Final LLM composition step that assembles the deliverable document.
- **Configuration choices:**  
  - Model: `models/gemini-2.5-flash`
  - Combines:
    - value match analysis from `Research Analyzer`
    - pitch assets from `Pitch Assets generator`
    - source URLs from `Company research`
  - Instructs markdown output suitable for a Google Docs writer flow
- **Key expressions or variables used:**  
  - `{{ $('Research Analyzer').item.json.content.parts.last().text }}`
  - `{{ $json.content.parts.last().text }}`
  - `{{ $('Company research').item.json.sources }}`
  - `{{ $('On form submission').item.json['Company Name'] }}`
- **Input and output connections:**  
  - Input: `Pitch Assets generator`
  - Output: `Markdown`
- **Version-specific requirements:**  
  `typeVersion: 1.1`
- **Edge cases or potential failure types:**  
  - Missing `sources` field from Olostep may produce blank source section
  - Markdown may not render ideally if Gemini returns inconsistent formatting
  - Token limitations if upstream content becomes too large
- **Sub-workflow reference:**  
  None

---

## 2.6 Markdown-to-Google-Doc Conversion

**Overview:**  
This block transforms the final markdown into HTML, converts that HTML into a binary file, uploads it to Google Drive, and duplicates it into native Google Docs format.

**Nodes Involved:**  
- Markdown
- html content
- Generate HTML Binary File
- Upload html file
- Transfer HTML to Doc

### Node Details

#### Markdown
- **Type and technical role:** `n8n-nodes-base.markdown`  
  Converts markdown text into HTML.
- **Configuration choices:**  
  - Mode: `markdownToHtml`
  - Options enabled:
    - tables
    - simplified auto-link
    - complete HTML document
  - Source markdown:
    - final Gemini battle card output
- **Key expressions or variables used:**  
  - `{{ $json.content.parts.last().text }}`
- **Input and output connections:**  
  - Input: `Sales battle card generator`
  - Output: `html content`
- **Version-specific requirements:**  
  `typeVersion: 1`
- **Edge cases or potential failure types:**  
  - If upstream text is not valid markdown, formatting may degrade
  - HTML output may not preserve all markdown nuances in Google Docs conversion
- **Sub-workflow reference:**  
  None

#### html content
- **Type and technical role:** `n8n-nodes-base.set`  
  Simple normalization node that copies the HTML output into a `data` field.
- **Configuration choices:**  
  - Assigns string field `data = $json.data`
- **Key expressions or variables used:**  
  - `{{ $json.data }}`
- **Input and output connections:**  
  - Input: `Markdown`
  - Output: `Generate HTML Binary File`
- **Version-specific requirements:**  
  `typeVersion: 3.4`
- **Edge cases or potential failure types:**  
  - If markdown conversion output field name differs, this mapping fails
- **Sub-workflow reference:**  
  None

#### Generate HTML Binary File
- **Type and technical role:** `n8n-nodes-base.code`  
  Converts the HTML string into a base64 binary file for Drive upload.
- **Configuration choices:**  
  - JavaScript maps all incoming items
  - Uses first item’s `json.data` as HTML source
  - Creates binary field `htmlFile`
  - File name: `output.html`
  - MIME type: `text/html`
- **Key expressions or variables used:**  
  In code:
  - `$input.all()`
  - `$input.first().json.data`
  - `Buffer.from(html, 'utf8').toString('base64')`
- **Input and output connections:**  
  - Input: `html content`
  - Output: `Upload html file`
- **Version-specific requirements:**  
  `typeVersion: 2`  
  Requires n8n code node support with `Buffer` available.
- **Edge cases or potential failure types:**  
  - If no input item exists, `$input.first()` fails
  - If `data` is undefined, generated file will be invalid or empty
  - Comment text in code is partly Dutch (`Loop over alle items`, `jouw HTML-string`) but harmless
- **Sub-workflow reference:**  
  None

#### Upload html file
- **Type and technical role:** `n8n-nodes-base.googleDrive`  
  Uploads the generated HTML binary file to Google Drive.
- **Configuration choices:**  
  - Upload target: My Drive / root
  - Binary input field: `htmlFile`
  - File name parameter: `working isa`
- **Key expressions or variables used:**  
  No dynamic expression in the visible main parameters.
- **Input and output connections:**  
  - Input: `Generate HTML Binary File`
  - Output: `Transfer HTML to Doc`
- **Version-specific requirements:**  
  `typeVersion: 3`
- **Edge cases or potential failure types:**  
  - Google Drive auth issues
  - Insufficient upload permissions
  - Root folder restrictions in some Drive setups
  - Static file name may cause clutter or confusion
- **Sub-workflow reference:**  
  None

#### Transfer HTML to Doc
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Calls the Google Drive API directly to copy the uploaded HTML file into a Google Docs document.
- **Configuration choices:**  
  - Method: `POST`
  - URL:
    - `https://www.googleapis.com/drive/v3/files/{{ $json.id }}/copy?fields=id,name,mimeType,webViewLink`
  - Authentication: predefined credential type
  - Credential type: `googleDriveOAuth2Api`
  - JSON body sets:
    - document name: `Sales Battle Card for [Company Name]`
    - MIME type: `application/vnd.google-apps.document`
- **Key expressions or variables used:**  
  - `{{ $json.id }}`
  - `{{ $('On form submission').item.json['Company Name'] }}`
- **Input and output connections:**  
  - Input: `Upload html file`
  - Output: `Delete html file`
- **Version-specific requirements:**  
  `typeVersion: 4.2`
- **Edge cases or potential failure types:**  
  - Google Drive OAuth scope may be insufficient for copy operations
  - API copy behavior may not perfectly preserve HTML formatting
  - If upload fails and `id` is missing, request URL becomes invalid
- **Sub-workflow reference:**  
  None

---

## 2.7 Cleanup, Sharing, and Notification

**Overview:**  
This block removes the temporary uploaded HTML file and shares the newly created Google Doc with a specified recipient, optionally sending a notification email.

**Nodes Involved:**  
- Delete html file
- Share link & email

### Node Details

#### Delete html file
- **Type and technical role:** `n8n-nodes-base.googleDrive`  
  Deletes the temporary HTML file after the Google Docs copy is created.
- **Configuration choices:**  
  - Operation: `deleteFile`
  - File ID taken from the original upload node
- **Key expressions or variables used:**  
  - `{{ $('Upload html file').item.json.id }}`
- **Input and output connections:**  
  - Input: `Transfer HTML to Doc`
  - Output: `Share link & email`
- **Version-specific requirements:**  
  `typeVersion: 3`
- **Edge cases or potential failure types:**  
  - If upload node did not return an ID, deletion fails
  - If file already removed manually, Google API returns not found
  - Failure here may prevent the share step because it is in sequence
- **Sub-workflow reference:**  
  None

#### Share link & email
- **Type and technical role:** `n8n-nodes-base.googleDrive`  
  Shares the generated Google Doc with a user email and includes a message.
- **Configuration choices:**  
  - Operation: `share`
  - File ID from `Transfer HTML to Doc`
  - Permission:
    - role: `reader`
    - type: `user`
    - email: `user@example.com`
  - Email message includes company-specific document name
- **Key expressions or variables used:**  
  - `{{ $('Transfer HTML to Doc').item.json.id }}`
  - `{{ $('On form submission').item.json['Company Name'] }}`
- **Input and output connections:**  
  - Input: `Delete html file`
  - Output: none
- **Version-specific requirements:**  
  `typeVersion: 3`
- **Edge cases or potential failure types:**  
  - Placeholder email address must be replaced
  - Google Drive email notification behavior depends on account/domain settings
  - External sharing restrictions may block the permission creation
- **Sub-workflow reference:**  
  None

---

## 2.8 Documentation and Visual Annotation Nodes

**Overview:**  
These nodes are non-executable sticky notes used to explain the workflow visually in the canvas. They do not affect execution but are important for understanding intent and grouped behavior.

**Nodes Involved:**  
- Sticky Note
- Sticky Note1
- Sticky Note2
- Sticky Note3
- Sticky Note4
- Sticky Note5
- Sticky Note6
- Sticky Note7
- Sticky Note8

### Node Details

#### Sticky Note
- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Canvas documentation block describing the overall workflow, target users, setup, customization, and requirements.
- **Configuration choices:**  
  Large informational note covering the full process.
- **Key expressions or variables used:**  
  None
- **Input and output connections:**  
  None
- **Version-specific requirements:**  
  `typeVersion: 1`
- **Edge cases or potential failure types:**  
  None; non-executable
- **Sub-workflow reference:**  
  None

#### Sticky Note1 to Sticky Note8
- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Section labels for visual grouping.
- **Configuration choices:**  
  Colored notes labeling input, research, AI analysis, pitch assets, battle card generation, markdown conversion, upload, and sharing.
- **Key expressions or variables used:**  
  None
- **Input and output connections:**  
  None
- **Version-specific requirements:**  
  `typeVersion: 1`
- **Edge cases or potential failure types:**  
  None
- **Sub-workflow reference:**  
  None

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| On form submission | `n8n-nodes-base.formTrigger` | Captures company name, website URL, and value proposition via form |  | Company research | ## Input Form  Enter company name, website, and your value proposition. |
| Company research | `n8n-nodes-olostep.olostepScrape` | Researches latest news, challenges, and hiring signals about the prospect company | On form submission | Scrape company website | ## Company Research  Olostep finds news, hiring trends, and business signals. Then scrapes the company website |
| Scrape company website | `n8n-nodes-olostep.olostepScrape` | Scrapes the company website for positioning and messaging content | Company research | Research Analyzer | ## Company Research  Olostep finds news, hiring trends, and business signals. Then scrapes the company website |
| Research Analyzer | `@n8n/n8n-nodes-langchain.googleGemini` | Identifies business triggers and maps the value proposition to prospect needs | Scrape company website | Pitch Assets generator | ## AI Analysis  Identifies business triggers and matches them with your value. |
| Pitch Assets generator | `@n8n/n8n-nodes-langchain.googleGemini` | Generates opener, discovery questions, and objection handling | Research Analyzer | Sales battle card generator | ## Pitch Assets  Generates outreach hook, questions, and objection handling. |
| Sales battle card generator | `@n8n/n8n-nodes-langchain.googleGemini` | Produces the final markdown sales battle card document | Pitch Assets generator | Markdown | ## Battle Card  Combines everything into a structured sales document. |
| Markdown | `n8n-nodes-base.markdown` | Converts the markdown battle card into HTML | Sales battle card generator | html content | ## Markdown to HTML Converts Markdown content into HTML and creates an HTML binary file. |
| html content | `n8n-nodes-base.set` | Copies HTML into a field for binary conversion | Markdown | Generate HTML Binary File | ## Markdown to HTML Converts Markdown content into HTML and creates an HTML binary file. |
| Generate HTML Binary File | `n8n-nodes-base.code` | Creates an HTML binary file from the generated HTML | html content | Upload html file | ## Markdown to HTML Converts Markdown content into HTML and creates an HTML binary file. |
| Upload html file | `n8n-nodes-base.googleDrive` | Uploads the temporary HTML file to Google Drive | Generate HTML Binary File | Transfer HTML to Doc | ## Upload The File Uploads the HTML file then converts it to google doc format and re-upload the file. And then deletes the old HTML file. |
| Transfer HTML to Doc | `n8n-nodes-base.httpRequest` | Copies the uploaded HTML file into native Google Docs format via Drive API | Upload html file | Delete html file | ## Upload The File Uploads the HTML file then converts it to google doc format and re-upload the file. And then deletes the old HTML file. |
| Delete html file | `n8n-nodes-base.googleDrive` | Deletes the temporary uploaded HTML file | Transfer HTML to Doc | Share link & email | ## Upload The File Uploads the HTML file then converts it to google doc format and re-upload the file. And then deletes the old HTML file. |
| Share link & email | `n8n-nodes-base.googleDrive` | Shares the final Google Doc and sends a share message by email | Delete html file |  | ## Share & Notify Shares the doc and sends an email with the link. |
| Sticky Note | `n8n-nodes-base.stickyNote` | Canvas documentation for workflow purpose, setup, and customization |  |  | # AI Sales Battle Card Generator (Research + Personalized Pitch Assets)  This n8n template automatically generates **high-quality sales battle cards** for any prospect company using real-time research and AI.  It analyzes company news, website content, and your product’s value proposition to create **personalized outreach hooks, talk tracks, and objection handling** — all delivered as a ready-to-use document.  ## Who’s it for  - Sales reps and SDRs preparing for outreach  - Founders doing high-quality cold outreach  - Sales teams building personalized pitches at scale  - Agencies doing outbound lead generation  - Anyone who wants deeper, research-driven sales conversations  ## How it works / What it does  1. **Form Input**  - Enter:  - Company name  - Company website URL  - Your product’s value proposition  2. **Company Research (Olostep)**  - Searches for the latest company news, hiring trends, and challenges.  - Identifies signals like growth, funding, or operational pressure.  3. **Website Scraping**  - Scrapes the company’s website to understand positioning, messaging, and offerings.  4. **AI Research Analysis**  - AI identifies:  - Key business triggers (why now?)  - Opportunities where your product fits  - Maps your value proposition to real company needs.  5. **Pitch Asset Generation**  - Generates personalized sales assets:  - Cold outreach hook (email/LinkedIn opener)  - High-impact discovery questions  - Likely objection + smart rebuttal  6. **Battle Card Generation**  - Combines everything into a structured **Sales Battle Card** including:  - Executive summary  - Outreach assets  - Objection handling  - Research sources  7. **Document Creation (Google Docs)**  - Converts the output into a clean document.  - Automatically creates a Google Doc with the final battle card.  8. **Share & Notify**  - Shares the document via Google Drive.  - Sends an email with the link to the generated battle card.  This workflow turns raw company data into a **ready-to-use sales playbook in minutes**.  ## How to set up  1. Import the template into your n8n workspace.  2. Add your **Olostep API key**.  3. Connect your **Google Drive** account.  4. Add your **AI model provider** (Gemini or OpenAI).  5. Run the form and generate your first battle card.  ## Requirements  - n8n account (cloud or self-hosted)  - Olostep API key  - Google Drive / Google Docs access  - AI model provider (Gemini or OpenAI)  ## How to customize the workflow  - Adjust prompts to match your sales style or industry.  - Add CRM integration (HubSpot, Salesforce).  - Generate multiple variations of outreach hooks.  - Add scoring for lead qualification.  - Store battle cards in Notion or a database.  ---  This template helps you go from generic outreach to highly personalized, research-driven sales conversations instantly. |
| Sticky Note1 | `n8n-nodes-base.stickyNote` | Visual label for input section |  |  | ## Input Form  Enter company name, website, and your value proposition. |
| Sticky Note2 | `n8n-nodes-base.stickyNote` | Visual label for research section |  |  | ## Company Research  Olostep finds news, hiring trends, and business signals. Then scrapes the company website |
| Sticky Note3 | `n8n-nodes-base.stickyNote` | Visual label for AI analysis section |  |  | ## AI Analysis  Identifies business triggers and matches them with your value. |
| Sticky Note4 | `n8n-nodes-base.stickyNote` | Visual label for pitch assets section |  |  | ## Pitch Assets  Generates outreach hook, questions, and objection handling. |
| Sticky Note5 | `n8n-nodes-base.stickyNote` | Visual label for battle card section |  |  | ## Battle Card  Combines everything into a structured sales document. |
| Sticky Note6 | `n8n-nodes-base.stickyNote` | Visual label for sharing section |  |  | ## Share & Notify Shares the doc and sends an email with the link. |
| Sticky Note7 | `n8n-nodes-base.stickyNote` | Visual label for markdown conversion section |  |  | ## Markdown to HTML Converts Markdown content into HTML and creates an HTML binary file. |
| Sticky Note8 | `n8n-nodes-base.stickyNote` | Visual label for upload/conversion section |  |  | ## Upload The File Uploads the HTML file then converts it to google doc format and re-upload the file. And then deletes the old HTML file. |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it something like: `Sales Battle Card`
   - Optional workflow settings from the source:
     - Binary mode: `separate`
     - Execution order: `v1`

2. **Add a Form Trigger node**
   - Node type: `Form Trigger`
   - Title: `Sales Battle Card for a prospect`
   - Description: `Please enter the company information`
   - Add fields:
     1. `Company Name`
     2. `Company website URL`
     3. `Value proposition` with field type `Textarea`
   - This is the entry point.

3. **Add an Olostep scrape/research node for company research**
   - Node type: `Olostep Scrape`
   - Connect it from `On form submission`
   - Configure credentials with your `Olostep Scrape account`
   - Set resource/mode to return an `answer`
   - Set the task prompt to:
     - `Find me the newest {{ $json['Company Name'] }} news, {{ $json['Company Name'] }} challenges, {{ $json['Company Name'] }} hiring.`
   - Note: the original workflow says `newst`; use `newest` if you want cleaner prompting.

4. **Add a second Olostep scrape node for website content**
   - Node type: `Olostep Scrape`
   - Connect it from `Company research`
   - Set `url_to_scrape` to:
     - `{{ $('On form submission').item.json['Company website URL'] }}`
   - This should return website content such as markdown or page text.

5. **Add a Gemini node for research analysis**
   - Node type: `Google Gemini`
   - Connect it from `Scrape company website`
   - Configure Gemini credentials
   - Use model:
     - `models/gemini-2.5-flash`
   - Enable `JSON Output`
   - Add prompt messages:
     - First message content:
       - `Research Data: {{ $('Company research').item.json.result.result }}`
       - `Value Proposition: {{ $('On form submission').item.json['Value proposition'] }}`
     - Second instruction message:
       - Tell the model it is a professional research analyzer
       - Ask it to identify 2–3 critical business triggers
       - Ask it to explain how the product solves the pressure created by those triggers
   - Keep output structured enough for downstream reuse.

6. **Add a Gemini node for pitch asset generation**
   - Node type: `Google Gemini`
   - Connect it from `Research Analyzer`
   - Use the same Gemini credentials
   - Use model:
     - `models/gemini-2.5-flash`
   - Prompt should combine:
     - prior analysis: `{{ $json.content.parts.last().text }}`
     - website content: `{{ $('Scrape company website').item.json.markdown_content }}`
   - Instruct the model to generate:
     - a 2-sentence outreach hook
     - 2 value-based questions
     - a predicted objection and one-sentence rebuttal
   - Request clean Markdown output only.

7. **Add a Gemini node for final battle card assembly**
   - Node type: `Google Gemini`
   - Connect it from `Pitch Assets generator`
   - Use the same Gemini credentials
   - Use model:
     - `models/gemini-2.5-flash`
   - Build the prompt with:
     - `{{ $('Research Analyzer').item.json.content.parts.last().text }}`
     - `{{ $json.content.parts.last().text }}`
     - `{{ $('Company research').item.json.sources }}`
     - `{{ $('On form submission').item.json['Company Name'] }}`
   - Instruct the model to output a markdown battle card with sections:
     - Sales Battle Card: [Company Name]
     - Executive Summary
     - Outreach Assets
     - Objection Handling
     - Research Sources
   - Request no meta-commentary.

8. **Add a Markdown node**
   - Node type: `Markdown`
   - Connect it from `Sales battle card generator`
   - Mode: `Markdown to HTML`
   - Source markdown:
     - `{{ $json.content.parts.last().text }}`
   - Enable options:
     - tables
     - simplified auto-link
     - complete HTML document

9. **Add a Set node named `html content`**
   - Node type: `Set`
   - Connect it from `Markdown`
   - Create one string field:
     - Name: `data`
     - Value: `{{ $json.data }}`
   - This normalizes the HTML into a field used by the code node.

10. **Add a Code node to create a binary HTML file**
    - Node type: `Code`
    - Connect it from `html content`
    - Use JavaScript similar to:
      - read `json.data`
      - convert to base64
      - assign binary property `htmlFile`
      - set filename to `output.html`
      - set MIME type to `text/html`
    - The output should contain binary data ready for Drive upload.

11. **Add a Google Drive upload node**
    - Node type: `Google Drive`
    - Connect it from `Generate HTML Binary File`
    - Configure Google Drive OAuth2 credentials
    - Upload the binary field:
      - `htmlFile`
    - Drive: `My Drive`
    - Folder: root or another target folder of your choice
    - File name can be static as in the original (`working isa`) or replaced with a dynamic meaningful name.

12. **Add an HTTP Request node to convert the uploaded HTML file into a Google Doc**
    - Node type: `HTTP Request`
    - Connect it from `Upload html file`
    - Authentication:
      - Predefined credential type
      - Use `Google Drive OAuth2 API`
    - Method: `POST`
    - URL:
      - `https://www.googleapis.com/drive/v3/files/{{ $json.id }}/copy?fields=id,name,mimeType,webViewLink`
    - Send JSON body:
      - `name`: `Sales Battle Card for {{ $('On form submission').item.json['Company Name'] }}`
      - `mimeType`: `application/vnd.google-apps.document`
    - This duplicates the uploaded HTML into native Google Docs format.

13. **Add a Google Drive delete node**
    - Node type: `Google Drive`
    - Connect it from `Transfer HTML to Doc`
    - Operation: `Delete File`
    - File ID:
      - `{{ $('Upload html file').item.json.id }}`
    - This removes the temporary HTML file once the Google Doc exists.

14. **Add a Google Drive share node**
    - Node type: `Google Drive`
    - Connect it from `Delete html file`
    - Operation: `Share`
    - File ID:
      - `{{ $('Transfer HTML to Doc').item.json.id }}`
    - Add permission:
      - role: `reader`
      - type: `user`
      - email address: replace `user@example.com` with the actual recipient
    - Optional email message:
      - `Hey,`
      - `Your "Sales Battle Card for {{ $('On form submission').item.json['Company Name'] }}" file is ready, find it at this link.`
    - This both shares the file and may trigger a Google-generated notification email depending on account settings.

15. **Add optional sticky notes for maintainability**
    - Add section labels if desired:
      - Input Form
      - Company Research
      - AI Analysis
      - Pitch Assets
      - Battle Card
      - Markdown to HTML
      - Upload The File
      - Share & Notify
    - Add one large note describing the workflow purpose, requirements, and customization options.

16. **Configure credentials**
    - **Olostep Scrape account**
      - Required for both Olostep nodes
    - **Gemini / Google PaLM API**
      - Required for all three Gemini nodes
    - **Google Drive OAuth2**
      - Required for upload, delete, share, and HTTP copy request
      - Ensure scopes allow file creation, copy, deletion, and sharing

17. **Test with a known company**
    - Submit:
      - a valid company name
      - a publicly accessible website
      - a clear value proposition
    - Verify:
      - Olostep returns research and sources
      - website scrape returns markdown/text
      - Gemini returns content in `content.parts[].text`
      - HTML uploads successfully
      - Drive copy creates a Google Doc
      - sharing completes for the recipient

18. **Recommended hardening improvements**
    - Replace static share email with a form field or environment variable
    - Add validation for website URL
    - Add error handling branches after:
      - Olostep nodes
      - Gemini nodes
      - Drive/API operations
    - Consider a fallback if `sources` or `markdown_content` is missing
    - Use a more descriptive uploaded HTML filename

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| This workflow creates research-driven AI sales battle cards using Olostep, Gemini, and Google Docs. | Overall workflow purpose |
| Intended audiences include sales reps, SDRs, founders, sales teams, and outbound agencies. | Workflow positioning |
| Main setup requirements: n8n account, Olostep API key, Google Drive/Docs access, and an AI model provider such as Gemini or OpenAI. | Setup prerequisites |
| Customization ideas mentioned in the workflow notes: adjust prompts, add CRM integrations like HubSpot or Salesforce, generate multiple hook variants, add lead scoring, or store outputs in Notion/database tools. | Extension ideas |
| The workflow note mentions OpenAI as an alternative model provider, but the actual implementation uses Gemini nodes only. | Implementation vs note mismatch |
| The document delivery method is Google Drive sharing, not a dedicated email node. Notification depends on Google Drive share behavior. | Important operational detail |
| The temporary file conversion relies on Google Drive API copy to `application/vnd.google-apps.document`. Formatting fidelity may vary depending on the HTML content. | Google Docs conversion behavior |