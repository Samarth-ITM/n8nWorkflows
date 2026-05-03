Generate and send AI newsletters with OpenAI, Tavily, Google Sheets and Gmail

https://n8nworkflows.xyz/workflows/generate-and-send-ai-newsletters-with-openai--tavily--google-sheets-and-gmail-14412


# Generate and send AI newsletters with OpenAI, Tavily, Google Sheets and Gmail

## 1. Workflow Overview

This workflow automates the full production and distribution of an AI-generated newsletter. A user submits a form with a brand name, brand website, and a news query. The workflow then:

1. fetches recent news on the requested topic,
2. analyzes the brand website to infer logo and color theme,
3. uses OpenAI to generate a newsletter title and three topic angles,
4. researches each topic and writes newsletter sections,
5. merges the sections into branded HTML,
6. stores the generated newsletter in Google Sheets,
7. sends it to all subscribers via Gmail.

It is designed for content teams, founders, agencies, and automation builders who want a repeatable branded newsletter pipeline with minimal manual writing.

### 1.1 Input Reception
This block receives the newsletter generation request from an n8n form. It captures the brand and topic inputs that drive the rest of the workflow.

### 1.2 Brand Identity Extraction
This block fetches the brand website twice and uses AI to extract:
- a likely logo URL,
- a color palette with primary, secondary, accent, and background colors.

These outputs are later injected into the final HTML email.

### 1.3 News Retrieval and Topic Planning
This block searches Tavily for recent news based on the submitted query, then uses AI to create:
- a short newsletter title,
- exactly three topic prompts.

### 1.4 Topic-by-Topic Research and Section Writing
Each of the three generated topics is processed independently. Tavily is queried again per topic, and OpenAI writes one newsletter section for each.

### 1.5 Content Consolidation and Subscriber Loading
The workflow merges:
- extracted logo,
- extracted brand colors,
- generated newsletter sections.

It then retrieves subscriber rows from Google Sheets and prepares batch iteration.

### 1.6 HTML Composition, Archiving, and Email Delivery
For each subscriber, AI composes the final branded HTML newsletter and subject line, saves a draft record into Google Sheets, and sends the email via Gmail.

---

## 2. Block-by-Block Analysis

## 2.1 Input Reception

**Overview:**  
This block exposes a form endpoint that starts the workflow. It collects the minimum required information needed to create a branded newsletter.

**Nodes Involved:**  
- On form submission

### Node Details

#### On form submission
- **Type and technical role:** `n8n-nodes-base.formTrigger`  
  Entry-point trigger node that starts the workflow via an n8n-hosted form.
- **Configuration choices:**  
  - Form title: `Newsletter`
  - Required fields:
    - `Brand Name`
    - `Brand Website`
    - `Query`
- **Key expressions or variables used:**  
  Outputs fields such as:
  - `$json['Brand Name']`
  - `$json['Brand Website']`
  - `$json['Query']`
- **Input and output connections:**  
  - No input; this is the trigger.
  - Outputs to:
    - `HTTP Request`
    - `HTTP Request2`
    - `HTTP Request6`
- **Version-specific requirements:**  
  Type version `2.3`; requires n8n version supporting Form Trigger v2.x.
- **Edge cases or potential failure types:**  
  - Invalid or unreachable brand website submitted by user.
  - Empty or vague query leading to weak Tavily results.
  - Public form misuse/spam if the endpoint is exposed without protections.
- **Sub-workflow reference:**  
  None.

---

## 2.2 Brand Identity Extraction

**Overview:**  
This block visits the submitted brand website and uses AI to infer visual branding elements. One AI path extracts a logo URL, while another extracts a normalized color theme.

**Nodes Involved:**  
- HTTP Request2
- AI Agent
- Structured Output Parser2
- OpenAI Chat Model4
- HTTP Request6
- Information Extractor1

### Node Details

#### HTTP Request2
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Fetches the brand website HTML for color/theme analysis.
- **Configuration choices:**  
  - URL is dynamically set to the submitted website:
    - `={{ $json['Brand Website'] }}`
- **Key expressions or variables used:**  
  - `$json['Brand Website']`
- **Input and output connections:**  
  - Input: `On form submission`
  - Output: `AI Agent`
- **Version-specific requirements:**  
  Type version `4.2`.
- **Edge cases or potential failure types:**  
  - DNS/TLS errors.
  - 403/anti-bot protections.
  - Redirect loops.
  - Website returns script-heavy HTML with little usable branding data.
- **Sub-workflow reference:**  
  None.

#### AI Agent
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent`  
  Uses an LLM to analyze website HTML and infer a brand palette.
- **Configuration choices:**  
  - Prompt input text: website response body via `={{ $json.data }}`
  - System instruction asks for:
    - `primary_color`
    - `secondary_color`
    - `accent_color`
    - `background_color`
  - Structured output enabled.
- **Key expressions or variables used:**  
  - `$json.data`
- **Input and output connections:**  
  - Main input: `HTTP Request2`
  - AI language model input: `OpenAI Chat Model4`
  - AI output parser input: `Structured Output Parser2`
  - Main output: `Merge` on input 1
- **Version-specific requirements:**  
  Type version `2.2`; requires compatible LangChain agent support in n8n.
- **Edge cases or potential failure types:**  
  - HTML too noisy or too large for useful inference.
  - Model returns invalid color strings unless parser constrains it.
  - Website theme may be misinterpreted if multiple stylesheets/brands appear.
- **Sub-workflow reference:**  
  None.

#### Structured Output Parser2
- **Type and technical role:** `@n8n/n8n-nodes-langchain.outputParserStructured`  
  Forces the AI color analysis into a structured JSON object.
- **Configuration choices:**  
  Example schema includes:
  - `primary_color`
  - `secondary_color`
  - `accent_color`
  - `background_color`
- **Key expressions or variables used:**  
  None beyond schema enforcement.
- **Input and output connections:**  
  - Connected to `AI Agent` as AI output parser
- **Version-specific requirements:**  
  Type version `1.3`.
- **Edge cases or potential failure types:**  
  - Parser failure if model output does not match schema.
  - Empty strings may still pass semantically but provide poor branding output.
- **Sub-workflow reference:**  
  None.

#### OpenAI Chat Model4
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
  Shared OpenAI model used by both website-analysis AI nodes.
- **Configuration choices:**  
  - Model: `gpt-4.1-mini`
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  - AI language model for:
    - `Information Extractor1`
    - `AI Agent`
- **Version-specific requirements:**  
  Type version `1.2`; requires OpenAI credentials configured in n8n.
- **Edge cases or potential failure types:**  
  - OpenAI auth/rate-limit errors.
  - Model context limits with large HTML pages.
- **Sub-workflow reference:**  
  None.

#### HTTP Request6
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Fetches the brand website HTML for logo extraction.
- **Configuration choices:**  
  - URL:
    - `={{ $json['Brand Website'] }}`
- **Key expressions or variables used:**  
  - `$json['Brand Website']`
- **Input and output connections:**  
  - Input: `On form submission`
  - Output: `Information Extractor1`
- **Version-specific requirements:**  
  Type version `4.2`.
- **Edge cases or potential failure types:**  
  Same as `HTTP Request2`; this duplication doubles the number of site fetches and potential failures.
- **Sub-workflow reference:**  
  None.

#### Information Extractor1
- **Type and technical role:** `@n8n/n8n-nodes-langchain.informationExtractor`  
  Uses the LLM to extract a logo image link from website HTML.
- **Configuration choices:**  
  - Input text: `={{ $json.data }}`
  - Extracted attribute:
    - `Logo` described as `Logo Image Link`
- **Key expressions or variables used:**  
  - `$json.data`
- **Input and output connections:**  
  - Main input: `HTTP Request6`
  - AI language model: `OpenAI Chat Model4`
  - Main output: `Merge` on input 0
- **Version-specific requirements:**  
  Type version `1.2`.
- **Edge cases or potential failure types:**  
  - Model may extract favicon or non-logo image.
  - Relative image URLs may be returned instead of absolute URLs.
  - Websites with multiple logos may create ambiguity.
- **Sub-workflow reference:**  
  None.

---

## 2.3 News Retrieval and Topic Planning

**Overview:**  
This block fetches current news from Tavily using the submitted query, then asks OpenAI to propose a concise title and exactly three newsletter topics.

**Nodes Involved:**  
- HTTP Request
- Generate Draft Topics (AI)
- Structured Output Parser
- OpenAI Chat Model
- Split Topics

### Node Details

#### HTTP Request
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Calls Tavily Search API for recent news on the submitted query.
- **Configuration choices:**  
  - Method: `POST`
  - URL: `https://api.tavily.com/search`
  - Sends JSON body:
    - `query`: submitted form query
    - `include_answer`: `advanced`
    - `topic`: `news`
    - `search_depth`: `advanced`
    - `time_range`: `week`
  - Sends Authorization header with bearer token placeholder
- **Key expressions or variables used:**  
  - `{{ $json.Query }}`
- **Input and output connections:**  
  - Input: `On form submission`
  - Output: `Generate Draft Topics (AI)`
- **Version-specific requirements:**  
  Type version `4.2`.
- **Edge cases or potential failure types:**  
  - Missing or invalid Tavily API token.
  - Search returns irrelevant or sparse results.
  - Quota/rate limit errors.
- **Sub-workflow reference:**  
  None.

#### Generate Draft Topics (AI)
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent`  
  Transforms Tavily search results into a newsletter title and three concise topic prompts.
- **Configuration choices:**  
  - Prompt text serializes Tavily results:
    - `={{ $json.results.map(item => JSON.stringify(item,null,5)).join('\n\\n') }}`
  - System message defines the model as an expert newsletter planner.
  - Requires a creative title and exactly three topics.
  - Structured output enabled.
- **Key expressions or variables used:**  
  - `$json.results`
- **Input and output connections:**  
  - Main input: `HTTP Request`
  - AI language model: `OpenAI Chat Model`
  - AI output parser: `Structured Output Parser`
  - Main output: `Split Topics`
- **Version-specific requirements:**  
  Type version `2.2`.
- **Edge cases or potential failure types:**  
  - If Tavily response lacks `results`, expression may fail.
  - Poor source diversity can lead to repetitive topics.
  - Parser may reject output if not exactly three topics.
- **Sub-workflow reference:**  
  None.

#### Structured Output Parser
- **Type and technical role:** `@n8n/n8n-nodes-langchain.outputParserStructured`  
  Validates that the AI planner returns a title and exactly three topics.
- **Configuration choices:**  
  Manual JSON schema:
  - object
  - required: `title`, `topics`
  - `topics` must be an array with exactly 3 string items
  - `additionalProperties: false`
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  - Connected to `Generate Draft Topics (AI)` as parser
- **Version-specific requirements:**  
  Type version `1.3`.
- **Edge cases or potential failure types:**  
  - Any extra field or fewer/more than 3 topics causes parsing failure.
- **Sub-workflow reference:**  
  None.

#### OpenAI Chat Model
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
  Supplies the LLM used for draft topic generation.
- **Configuration choices:**  
  - Model: `gpt-4.1-mini`
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  - AI language model for `Generate Draft Topics (AI)`
- **Version-specific requirements:**  
  Type version `1.2`.
- **Edge cases or potential failure types:**  
  OpenAI credential, quota, or latency issues.
- **Sub-workflow reference:**  
  None.

#### Split Topics
- **Type and technical role:** `n8n-nodes-base.splitOut`  
  Splits the 3-topic array into separate items so each topic can be researched and written independently.
- **Configuration choices:**  
  - Field to split out: `output.topics`
- **Key expressions or variables used:**  
  - Reads the AI node’s structured output path `output.topics`
- **Input and output connections:**  
  - Input: `Generate Draft Topics (AI)`
  - Output: `Tavily`
- **Version-specific requirements:**  
  Type version `1`.
- **Edge cases or potential failure types:**  
  - If parser output shape changes, field path becomes invalid.
  - Empty topic array results in no downstream items.
- **Sub-workflow reference:**  
  None.

---

## 2.4 Topic-by-Topic Research and Section Writing

**Overview:**  
This block performs topic-level research and writing. Each generated topic is sent to Tavily, and AI writes one standalone newsletter section with citations.

**Nodes Involved:**  
- Tavily
- Generate Newsletter Content (AI)
- OpenAI Chat Model1
- Merge Content Pieces

### Node Details

#### Tavily
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Performs a Tavily search for each generated topic.
- **Configuration choices:**  
  - Method: implied HTTP request with body and headers
  - URL: `https://api.tavily.com/search`
  - Body parameter:
    - `query = {{ $json['output.topics'] }}`
  - Authorization bearer token placeholder
- **Key expressions or variables used:**  
  - `$json['output.topics']`
- **Input and output connections:**  
  - Input: `Split Topics`
  - Output: `Generate Newsletter Content (AI)`
- **Version-specific requirements:**  
  Type version `4.2`.
- **Edge cases or potential failure types:**  
  - Because `Split Topics` usually emits the split value, this expression may be fragile depending on resulting item shape. In some executions, the topic may need to be referenced differently.
  - Missing Tavily token or poor search results.
- **Sub-workflow reference:**  
  None.

#### Generate Newsletter Content (AI)
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent`  
  Writes one self-contained newsletter section for each researched topic.
- **Configuration choices:**  
  - Prompt text:
    - `Topic {{ $json.query }}`
    - followed by serialized research results
  - System message instructs:
    - standalone section only,
    - heading + content,
    - no intro/conclusion,
    - verifiable citations only,
    - clickable URLs,
    - concise expert tone.
- **Key expressions or variables used:**  
  - `$json.query`
  - `$json.results.map(...)`
- **Input and output connections:**  
  - Main input: `Tavily`
  - AI language model: `OpenAI Chat Model1`
  - Main output: `Merge Content Pieces`
- **Version-specific requirements:**  
  Type version `2.2`.
- **Edge cases or potential failure types:**  
  - If Tavily returns no `results`, the prompt can fail or become weak.
  - AI may still produce non-clickable citations unless carefully validated.
  - Hallucinated claims remain possible if the model over-summarizes sources.
- **Sub-workflow reference:**  
  None.

#### OpenAI Chat Model1
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
  LLM used to generate each section.
- **Configuration choices:**  
  - Model: `gpt-4.1-mini`
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  - AI language model for `Generate Newsletter Content (AI)`
- **Version-specific requirements:**  
  Type version `1.2`.
- **Edge cases or potential failure types:**  
  Same OpenAI authentication/rate-limit concerns.
- **Sub-workflow reference:**  
  None.

#### Merge Content Pieces
- **Type and technical role:** `n8n-nodes-base.aggregate`  
  Collects all generated section outputs into a single array.
- **Configuration choices:**  
  - Aggregates field: `output`
- **Key expressions or variables used:**  
  Works on AI node output field path.
- **Input and output connections:**  
  - Input: `Generate Newsletter Content (AI)`
  - Output: `Merge` on input 2
- **Version-specific requirements:**  
  Type version `1`.
- **Edge cases or potential failure types:**  
  - If fewer than three sections are generated, downstream composition may reference nonexistent array positions.
- **Sub-workflow reference:**  
  None.

---

## 2.5 Content Consolidation and Subscriber Loading

**Overview:**  
This block merges the brand assets and generated content, then fetches subscribers and prepares item-by-item email processing.

**Nodes Involved:**  
- Merge
- Aggregate
- Get row(s) in sheet
- Loop Over Items

### Node Details

#### Merge
- **Type and technical role:** `n8n-nodes-base.merge`  
  Combines three parallel branches:
  1. logo extraction,
  2. brand color extraction,
  3. generated newsletter sections.
- **Configuration choices:**  
  - Number of inputs: `3`
- **Key expressions or variables used:**  
  None directly.
- **Input and output connections:**  
  - Inputs:
    - `Information Extractor1`
    - `AI Agent`
    - `Merge Content Pieces`
  - Output: `Aggregate`
- **Version-specific requirements:**  
  Type version `3.2`.
- **Edge cases or potential failure types:**  
  - Uneven item counts or timing mismatches across branches.
  - Merge behavior depends on incoming item cardinality; if any branch fails, downstream structure may break.
- **Sub-workflow reference:**  
  None.

#### Aggregate
- **Type and technical role:** `n8n-nodes-base.aggregate`  
  Aggregates all merged items into a single `output` array for downstream AI HTML composition.
- **Configuration choices:**  
  - Aggregate mode: `aggregateAllItemData`
  - Destination field: `output`
- **Key expressions or variables used:**  
  Downstream nodes reference:
  - `$('Aggregate').item.json.output[0]`
  - `$('Aggregate').item.json.output[1]`
  - `$('Aggregate').item.json.output[2]`
- **Input and output connections:**  
  - Input: `Merge`
  - Output: `Get row(s) in sheet`
- **Version-specific requirements:**  
  Type version `1`.
- **Edge cases or potential failure types:**  
  - Downstream logic assumes stable ordering:
    - index 0 = logo
    - index 1 = colors
    - index 2 = section content
  - If item order changes, final email rendering breaks.
- **Sub-workflow reference:**  
  None.

#### Get row(s) in sheet
- **Type and technical role:** `n8n-nodes-base.googleSheets`  
  Retrieves subscribers from a Google Sheet.
- **Configuration choices:**  
  - Spreadsheet: `Newsletter Emails`
  - Sheet: `Subscribers List`
  - Operation appears to be row retrieval
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  - Input: `Aggregate`
  - Output: `Loop Over Items`
- **Version-specific requirements:**  
  Type version `4.7`; requires Google Sheets OAuth2 credentials.
- **Edge cases or potential failure types:**  
  - Missing credential scopes.
  - Wrong sheet tab selected.
  - Empty subscriber list yields no emails sent.
  - If rows do not contain `Email`, downstream Gmail node fails.
- **Sub-workflow reference:**  
  None.

#### Loop Over Items
- **Type and technical role:** `n8n-nodes-base.splitInBatches`  
  Iterates over subscriber rows to personalize and send emails one by one.
- **Configuration choices:**  
  - Default batch behavior
  - `executeOnce: true` is enabled, which may affect iteration semantics depending on n8n execution behavior
- **Key expressions or variables used:**  
  Downstream references subscriber fields through:
  - `$('Loop Over Items').item.json.Email`
- **Input and output connections:**  
  - Input: `Get row(s) in sheet`
  - Output 0: loop continuation from `Sending Emails to all the Subscribers`
  - Output 1: starts current-item processing at `Convert Newsletter to HTML (AI)`
- **Version-specific requirements:**  
  Type version `3`.
- **Edge cases or potential failure types:**  
  - Misunderstanding of output indexes can break loops.
  - `executeOnce: true` may reduce expected per-item behavior in some designs.
  - Rows missing email address will fail in Gmail node.
- **Sub-workflow reference:**  
  None.

---

## 2.6 HTML Composition, Archiving, and Email Delivery

**Overview:**  
This block generates the final branded HTML and subject line, logs the newsletter draft to Google Sheets, and sends it to each subscriber through Gmail.

**Nodes Involved:**  
- Convert Newsletter to HTML (AI)
- Structured Output Parser1
- OpenAI Chat Model2
- Save Newsletter Draft in Google Sheet
- Sending Emails to all the Subscribers

### Node Details

#### Convert Newsletter to HTML (AI)
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent`  
  Builds the final email subject and HTML body from title, sections, brand colors, logo, and subscriber context.
- **Configuration choices:**  
  - Prompt text includes:
    - title from `Generate Draft Topics (AI)`
    - merged sections from `Aggregate`
  - Extensive system prompt defines:
    - fixed HTML layout,
    - text-color-only theming,
    - dynamic logo,
    - date insertion,
    - citations formatting,
    - subject line length,
    - footer rules.
  - Structured output enabled.
- **Key expressions or variables used:**  
  - `$('Generate Draft Topics (AI)').item.json.output.title`
  - `$('Aggregate').item.json.output[2].output.join("\n\n\n")`
  - `$('Aggregate').item.json.output[1].output.secondary_color`
  - `$('Aggregate').item.json.output[0].output.Logo`
  - `$('On form submission').item.json['Brand Name']`
  - `$json.Name`
  - `$now.format('yyyy')`
  - `$now.format('yyyy-MM-dd')`
- **Input and output connections:**  
  - Main input: `Loop Over Items` output 1
  - AI language model: `OpenAI Chat Model2`
  - AI output parser: `Structured Output Parser1`
  - Main outputs:
    - `Save Newsletter Draft in Google Sheet`
    - `Sending Emails to all the Subscribers`
- **Version-specific requirements:**  
  Type version `2.2`.
- **Edge cases or potential failure types:**  
  - Downstream prompt assumes specific aggregate array indexes.
  - If subscriber row lacks `Name`, greeting becomes blank.
  - If logo/color extraction failed, HTML may contain empty attributes.
  - Prompt is long; token usage may rise significantly.
  - HTML returned by model may be malformed.
- **Sub-workflow reference:**  
  None.

#### Structured Output Parser1
- **Type and technical role:** `@n8n/n8n-nodes-langchain.outputParserStructured`  
  Enforces final AI output shape with subject and HTML content.
- **Configuration choices:**  
  Required object fields:
  - `subject`
  - `content`
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  - Connected to `Convert Newsletter to HTML (AI)` as parser
- **Version-specific requirements:**  
  Type version `1.3`.
- **Edge cases or potential failure types:**  
  - Parser will fail if the model returns plain text or malformed JSON-like output.
- **Sub-workflow reference:**  
  None.

#### OpenAI Chat Model2
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
  LLM used for final subject/body composition.
- **Configuration choices:**  
  - Model: `gpt-4.1-mini`
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  - AI language model for `Convert Newsletter to HTML (AI)`
- **Version-specific requirements:**  
  Type version `1.2`.
- **Edge cases or potential failure types:**  
  Same OpenAI availability, quota, and latency risks.
- **Sub-workflow reference:**  
  None.

#### Save Newsletter Draft in Google Sheet
- **Type and technical role:** `n8n-nodes-base.googleSheets`  
  Appends a record of the generated newsletter to a Google Sheet.
- **Configuration choices:**  
  - Operation: `append`
  - Spreadsheet: `Newsletter Emails`
  - Sheet: `Sheet1`
  - Mapped columns:
    - `Email` ← `{{ $json.output.subject }}`
    - `Message` ← `{{ $json.output.content }}`
    - `Email Send At` ← `{{$now}}`
    - `Email Created At` ← `{{$now}}`
- **Key expressions or variables used:**  
  - `$json.output.subject`
  - `$json.output.content`
  - `$now`
- **Input and output connections:**  
  - Input: `Convert Newsletter to HTML (AI)`
- **Version-specific requirements:**  
  Type version `4.7`; Google Sheets OAuth2 required.
- **Edge cases or potential failure types:**  
  - Column naming is misleading: subject is written into a column named `Email`.
  - Large HTML content may be awkward to store in sheet cells.
  - Append failures due to permissions or sheet limits.
- **Sub-workflow reference:**  
  None.

#### Sending Emails to all the Subscribers
- **Type and technical role:** `n8n-nodes-base.gmail`  
  Sends the generated HTML newsletter to the current subscriber.
- **Configuration choices:**  
  - Recipient:
    - `={{ $('Loop Over Items').item.json.Email }}`
  - Subject:
    - `={{ $('Convert Newsletter to HTML (AI)').item.json.output.subject }}`
  - Message:
    - `={{ $('Convert Newsletter to HTML (AI)').item.json.output.content }}`
- **Key expressions or variables used:**  
  - `$('Loop Over Items').item.json.Email`
  - `$('Convert Newsletter to HTML (AI)').item.json.output.subject`
  - `$('Convert Newsletter to HTML (AI)').item.json.output.content`
- **Input and output connections:**  
  - Input: `Convert Newsletter to HTML (AI)`
  - Output: back to `Loop Over Items` to continue iteration
- **Version-specific requirements:**  
  Type version `2.1`; Gmail OAuth2 required.
- **Edge cases or potential failure types:**  
  - Gmail account sending limits.
  - Invalid subscriber emails.
  - HTML may render as plain text if content type handling is not configured as expected in the node/account.
  - OAuth token expiration.
- **Sub-workflow reference:**  
  None.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| On form submission | n8n-nodes-base.formTrigger | Receives brand and query inputs from a form |  | HTTP Request; HTTP Request2; HTTP Request6 | # 📨 AI Newsletter Automation Workflow<br>## 🚀 Brief Overview (How It Works)<br>This workflow automates the complete process of generating and sending AI-powered newsletters. It starts with a form submission where the user provides a brand name, website, and topic query. The system then fetches relevant news, analyzes the brand’s visual identity (colors & logo), and uses AI to generate structured newsletter topics and content. Finally, it converts everything into a beautifully styled HTML email, saves it to Google Sheets, and sends it to all subscribers via Gmail.<br>## Quick Setup Guide<br>👉 [Demo & Setup Video](https://drive.google.com/file/d/1ulOGWNL3cKe3y7tSKWNG2jlpL7dyiAgX/view?usp=sharing)<br>👉 [Sheet Template](https://docs.google.com/spreadsheets/d/1hvB5Zif52eCLv_X7E_OifQv9OI5usn-CQ50-TsZTMQA/edit?usp=sharing)<br>👉 [Course](https://www.udemy.com/course/n8n-automation-mastery-build-ai-powered-enterprise-ready/?referralCode=2EAE71591D3BEB80F2CC)<br>## Workflow Input (Form Trigger)<br>This section collects the required inputs from the user through a form submission. |
| HTTP Request | n8n-nodes-base.httpRequest | Searches recent news in Tavily using submitted query | On form submission | Generate Draft Topics (AI) | # 📨 AI Newsletter Automation Workflow<br>## 🚀 Brief Overview (How It Works)<br>This workflow automates the complete process of generating and sending AI-powered newsletters. It starts with a form submission where the user provides a brand name, website, and topic query. The system then fetches relevant news, analyzes the brand’s visual identity (colors & logo), and uses AI to generate structured newsletter topics and content. Finally, it converts everything into a beautifully styled HTML email, saves it to Google Sheets, and sends it to all subscribers via Gmail.<br>## Quick Setup Guide<br>👉 [Demo & Setup Video](https://drive.google.com/file/d/1ulOGWNL3cKe3y7tSKWNG2jlpL7dyiAgX/view?usp=sharing)<br>👉 [Sheet Template](https://docs.google.com/spreadsheets/d/1hvB5Zif52eCLv_X7E_OifQv9OI5usn-CQ50-TsZTMQA/edit?usp=sharing)<br>👉 [Course](https://www.udemy.com/course/n8n-automation-mastery-build-ai-powered-enterprise-ready/?referralCode=2EAE71591D3BEB80F2CC)<br>## Fetch News About Query<br>This section retrieves news about the query |
| Generate Draft Topics (AI) | @n8n/n8n-nodes-langchain.agent | Creates newsletter title and exactly three topics | HTTP Request; OpenAI Chat Model; Structured Output Parser | Split Topics | # 📨 AI Newsletter Automation Workflow<br>## 🚀 Brief Overview (How It Works)<br>This workflow automates the complete process of generating and sending AI-powered newsletters. It starts with a form submission where the user provides a brand name, website, and topic query. The system then fetches relevant news, analyzes the brand’s visual identity (colors & logo), and uses AI to generate structured newsletter topics and content. Finally, it converts everything into a beautifully styled HTML email, saves it to Google Sheets, and sends it to all subscribers via Gmail.<br>## Quick Setup Guide<br>👉 [Demo & Setup Video](https://drive.google.com/file/d/1ulOGWNL3cKe3y7tSKWNG2jlpL7dyiAgX/view?usp=sharing)<br>👉 [Sheet Template](https://docs.google.com/spreadsheets/d/1hvB5Zif52eCLv_X7E_OifQv9OI5usn-CQ50-TsZTMQA/edit?usp=sharing)<br>👉 [Course](https://www.udemy.com/course/n8n-automation-mastery-build-ai-powered-enterprise-ready/?referralCode=2EAE71591D3BEB80F2CC)<br>## Fetch News About Query<br>This section retrieves news about the query |
| Structured Output Parser | @n8n/n8n-nodes-langchain.outputParserStructured | Validates title and three-topic AI output |  | Generate Draft Topics (AI) | # 📨 AI Newsletter Automation Workflow<br>## 🚀 Brief Overview (How It Works)<br>This workflow automates the complete process of generating and sending AI-powered newsletters. It starts with a form submission where the user provides a brand name, website, and topic query. The system then fetches relevant news, analyzes the brand’s visual identity (colors & logo), and uses AI to generate structured newsletter topics and content. Finally, it converts everything into a beautifully styled HTML email, saves it to Google Sheets, and sends it to all subscribers via Gmail.<br>## Quick Setup Guide<br>👉 [Demo & Setup Video](https://drive.google.com/file/d/1ulOGWNL3cKe3y7tSKWNG2jlpL7dyiAgX/view?usp=sharing)<br>👉 [Sheet Template](https://docs.google.com/spreadsheets/d/1hvB5Zif52eCLv_X7E_OifQv9OI5usn-CQ50-TsZTMQA/edit?usp=sharing)<br>👉 [Course](https://www.udemy.com/course/n8n-automation-mastery-build-ai-powered-enterprise-ready/?referralCode=2EAE71591D3BEB80F2CC)<br>## Fetch News About Query<br>This section retrieves news about the query |
| OpenAI Chat Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM for topic planning |  | Generate Draft Topics (AI) | # 📨 AI Newsletter Automation Workflow<br>## 🚀 Brief Overview (How It Works)<br>This workflow automates the complete process of generating and sending AI-powered newsletters. It starts with a form submission where the user provides a brand name, website, and topic query. The system then fetches relevant news, analyzes the brand’s visual identity (colors & logo), and uses AI to generate structured newsletter topics and content. Finally, it converts everything into a beautifully styled HTML email, saves it to Google Sheets, and sends it to all subscribers via Gmail.<br>## Quick Setup Guide<br>👉 [Demo & Setup Video](https://drive.google.com/file/d/1ulOGWNL3cKe3y7tSKWNG2jlpL7dyiAgX/view?usp=sharing)<br>👉 [Sheet Template](https://docs.google.com/spreadsheets/d/1hvB5Zif52eCLv_X7E_OifQv9OI5usn-CQ50-TsZTMQA/edit?usp=sharing)<br>👉 [Course](https://www.udemy.com/course/n8n-automation-mastery-build-ai-powered-enterprise-ready/?referralCode=2EAE71591D3BEB80F2CC)<br>## Fetch News About Query<br>This section retrieves news about the query |
| Split Topics | n8n-nodes-base.splitOut | Splits generated topics into individual items | Generate Draft Topics (AI) | Tavily | # 📨 AI Newsletter Automation Workflow<br>## 🚀 Brief Overview (How It Works)<br>This workflow automates the complete process of generating and sending AI-powered newsletters. It starts with a form submission where the user provides a brand name, website, and topic query. The system then fetches relevant news, analyzes the brand’s visual identity (colors & logo), and uses AI to generate structured newsletter topics and content. Finally, it converts everything into a beautifully styled HTML email, saves it to Google Sheets, and sends it to all subscribers via Gmail.<br>## Quick Setup Guide<br>👉 [Demo & Setup Video](https://drive.google.com/file/d/1ulOGWNL3cKe3y7tSKWNG2jlpL7dyiAgX/view?usp=sharing)<br>👉 [Sheet Template](https://docs.google.com/spreadsheets/d/1hvB5Zif52eCLv_X7E_OifQv9OI5usn-CQ50-TsZTMQA/edit?usp=sharing)<br>👉 [Course](https://www.udemy.com/course/n8n-automation-mastery-build-ai-powered-enterprise-ready/?referralCode=2EAE71591D3BEB80F2CC)<br>## Fetch News About Query<br>This section retrieves news about the query |
| Tavily | n8n-nodes-base.httpRequest | Searches Tavily for each generated topic | Split Topics | Generate Newsletter Content (AI) | # 📨 AI Newsletter Automation Workflow<br>## 🚀 Brief Overview (How It Works)<br>This workflow automates the complete process of generating and sending AI-powered newsletters. It starts with a form submission where the user provides a brand name, website, and topic query. The system then fetches relevant news, analyzes the brand’s visual identity (colors & logo), and uses AI to generate structured newsletter topics and content. Finally, it converts everything into a beautifully styled HTML email, saves it to Google Sheets, and sends it to all subscribers via Gmail.<br>## Quick Setup Guide<br>👉 [Demo & Setup Video](https://drive.google.com/file/d/1ulOGWNL3cKe3y7tSKWNG2jlpL7dyiAgX/view?usp=sharing)<br>👉 [Sheet Template](https://docs.google.com/spreadsheets/d/1hvB5Zif52eCLv_X7E_OifQv9OI5usn-CQ50-TsZTMQA/edit?usp=sharing)<br>👉 [Course](https://www.udemy.com/course/n8n-automation-mastery-build-ai-powered-enterprise-ready/?referralCode=2EAE71591D3BEB80F2CC)<br>## Fetch News About Query<br>This section retrieves news about the query |
| Generate Newsletter Content (AI) | @n8n/n8n-nodes-langchain.agent | Writes one cited section per topic | Tavily; OpenAI Chat Model1 | Merge Content Pieces | # 📨 AI Newsletter Automation Workflow<br>## 🚀 Brief Overview (How It Works)<br>This workflow automates the complete process of generating and sending AI-powered newsletters. It starts with a form submission where the user provides a brand name, website, and topic query. The system then fetches relevant news, analyzes the brand’s visual identity (colors & logo), and uses AI to generate structured newsletter topics and content. Finally, it converts everything into a beautifully styled HTML email, saves it to Google Sheets, and sends it to all subscribers via Gmail.<br>## Quick Setup Guide<br>👉 [Demo & Setup Video](https://drive.google.com/file/d/1ulOGWNL3cKe3y7tSKWNG2jlpL7dyiAgX/view?usp=sharing)<br>👉 [Sheet Template](https://docs.google.com/spreadsheets/d/1hvB5Zif52eCLv_X7E_OifQv9OI5usn-CQ50-TsZTMQA/edit?usp=sharing)<br>👉 [Course](https://www.udemy.com/course/n8n-automation-mastery-build-ai-powered-enterprise-ready/?referralCode=2EAE71591D3BEB80F2CC)<br>## Fetch News About Query<br>This section retrieves news about the query |
| OpenAI Chat Model1 | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM for section writing |  | Generate Newsletter Content (AI) | # 📨 AI Newsletter Automation Workflow<br>## 🚀 Brief Overview (How It Works)<br>This workflow automates the complete process of generating and sending AI-powered newsletters. It starts with a form submission where the user provides a brand name, website, and topic query. The system then fetches relevant news, analyzes the brand’s visual identity (colors & logo), and uses AI to generate structured newsletter topics and content. Finally, it converts everything into a beautifully styled HTML email, saves it to Google Sheets, and sends it to all subscribers via Gmail.<br>## Quick Setup Guide<br>👉 [Demo & Setup Video](https://drive.google.com/file/d/1ulOGWNL3cKe3y7tSKWNG2jlpL7dyiAgX/view?usp=sharing)<br>👉 [Sheet Template](https://docs.google.com/spreadsheets/d/1hvB5Zif52eCLv_X7E_OifQv9OI5usn-CQ50-TsZTMQA/edit?usp=sharing)<br>👉 [Course](https://www.udemy.com/course/n8n-automation-mastery-build-ai-powered-enterprise-ready/?referralCode=2EAE71591D3BEB80F2CC)<br>## Fetch News About Query<br>This section retrieves news about the query |
| Merge Content Pieces | n8n-nodes-base.aggregate | Collects all section outputs into an array | Generate Newsletter Content (AI) | Merge | # 📨 AI Newsletter Automation Workflow<br>## 🚀 Brief Overview (How It Works)<br>This workflow automates the complete process of generating and sending AI-powered newsletters. It starts with a form submission where the user provides a brand name, website, and topic query. The system then fetches relevant news, analyzes the brand’s visual identity (colors & logo), and uses AI to generate structured newsletter topics and content. Finally, it converts everything into a beautifully styled HTML email, saves it to Google Sheets, and sends it to all subscribers via Gmail.<br>## Quick Setup Guide<br>👉 [Demo & Setup Video](https://drive.google.com/file/d/1ulOGWNL3cKe3y7tSKWNG2jlpL7dyiAgX/view?usp=sharing)<br>👉 [Sheet Template](https://docs.google.com/spreadsheets/d/1hvB5Zif52eCLv_X7E_OifQv9OI5usn-CQ50-TsZTMQA/edit?usp=sharing)<br>👉 [Course](https://www.udemy.com/course/n8n-automation-mastery-build-ai-powered-enterprise-ready/?referralCode=2EAE71591D3BEB80F2CC)<br>## Fetch News About Query<br>This section retrieves news about the query |
| HTTP Request2 | n8n-nodes-base.httpRequest | Fetches website HTML for color analysis | On form submission | AI Agent | # 📨 AI Newsletter Automation Workflow<br>## 🚀 Brief Overview (How It Works)<br>This workflow automates the complete process of generating and sending AI-powered newsletters. It starts with a form submission where the user provides a brand name, website, and topic query. The system then fetches relevant news, analyzes the brand’s visual identity (colors & logo), and uses AI to generate structured newsletter topics and content. Finally, it converts everything into a beautifully styled HTML email, saves it to Google Sheets, and sends it to all subscribers via Gmail.<br>## Quick Setup Guide<br>👉 [Demo & Setup Video](https://drive.google.com/file/d/1ulOGWNL3cKe3y7tSKWNG2jlpL7dyiAgX/view?usp=sharing)<br>👉 [Sheet Template](https://docs.google.com/spreadsheets/d/1hvB5Zif52eCLv_X7E_OifQv9OI5usn-CQ50-TsZTMQA/edit?usp=sharing)<br>👉 [Course](https://www.udemy.com/course/n8n-automation-mastery-build-ai-powered-enterprise-ready/?referralCode=2EAE71591D3BEB80F2CC)<br>## Extract Brand Style & Colors<br>This block analyzes the provided brand website to determine the brand’s visual identity. |
| AI Agent | @n8n/n8n-nodes-langchain.agent | Infers brand colors from website HTML | HTTP Request2; OpenAI Chat Model4; Structured Output Parser2 | Merge | # 📨 AI Newsletter Automation Workflow<br>## 🚀 Brief Overview (How It Works)<br>This workflow automates the complete process of generating and sending AI-powered newsletters. It starts with a form submission where the user provides a brand name, website, and topic query. The system then fetches relevant news, analyzes the brand’s visual identity (colors & logo), and uses AI to generate structured newsletter topics and content. Finally, it converts everything into a beautifully styled HTML email, saves it to Google Sheets, and sends it to all subscribers via Gmail.<br>## Quick Setup Guide<br>👉 [Demo & Setup Video](https://drive.google.com/file/d/1ulOGWNL3cKe3y7tSKWNG2jlpL7dyiAgX/view?usp=sharing)<br>👉 [Sheet Template](https://docs.google.com/spreadsheets/d/1hvB5Zif52eCLv_X7E_OifQv9OI5usn-CQ50-TsZTMQA/edit?usp=sharing)<br>👉 [Course](https://www.udemy.com/course/n8n-automation-mastery-build-ai-powered-enterprise-ready/?referralCode=2EAE71591D3BEB80F2CC)<br>## Extract Brand Style & Colors<br>This block analyzes the provided brand website to determine the brand’s visual identity. |
| Structured Output Parser2 | @n8n/n8n-nodes-langchain.outputParserStructured | Structures brand color output |  | AI Agent | # 📨 AI Newsletter Automation Workflow<br>## 🚀 Brief Overview (How It Works)<br>This workflow automates the complete process of generating and sending AI-powered newsletters. It starts with a form submission where the user provides a brand name, website, and topic query. The system then fetches relevant news, analyzes the brand’s visual identity (colors & logo), and uses AI to generate structured newsletter topics and content. Finally, it converts everything into a beautifully styled HTML email, saves it to Google Sheets, and sends it to all subscribers via Gmail.<br>## Quick Setup Guide<br>👉 [Demo & Setup Video](https://drive.google.com/file/d/1ulOGWNL3cKe3y7tSKWNG2jlpL7dyiAgX/view?usp=sharing)<br>👉 [Sheet Template](https://docs.google.com/spreadsheets/d/1hvB5Zif52eCLv_X7E_OifQv9OI5usn-CQ50-TsZTMQA/edit?usp=sharing)<br>👉 [Course](https://www.udemy.com/course/n8n-automation-mastery-build-ai-powered-enterprise-ready/?referralCode=2EAE71591D3BEB80F2CC)<br>## Extract Brand Style & Colors<br>This block analyzes the provided brand website to determine the brand’s visual identity. |
| OpenAI Chat Model4 | @n8n/n8n-nodes-langchain.lmChatOpenAi | Shared LLM for logo and color extraction |  | Information Extractor1; AI Agent | # 📨 AI Newsletter Automation Workflow<br>## 🚀 Brief Overview (How It Works)<br>This workflow automates the complete process of generating and sending AI-powered newsletters. It starts with a form submission where the user provides a brand name, website, and topic query. The system then fetches relevant news, analyzes the brand’s visual identity (colors & logo), and uses AI to generate structured newsletter topics and content. Finally, it converts everything into a beautifully styled HTML email, saves it to Google Sheets, and sends it to all subscribers via Gmail.<br>## Quick Setup Guide<br>👉 [Demo & Setup Video](https://drive.google.com/file/d/1ulOGWNL3cKe3y7tSKWNG2jlpL7dyiAgX/view?usp=sharing)<br>👉 [Sheet Template](https://docs.google.com/spreadsheets/d/1hvB5Zif52eCLv_X7E_OifQv9OI5usn-CQ50-TsZTMQA/edit?usp=sharing)<br>👉 [Course](https://www.udemy.com/course/n8n-automation-mastery-build-ai-powered-enterprise-ready/?referralCode=2EAE71591D3BEB80F2CC)<br>## Extract Brand Style & Colors<br>This block analyzes the provided brand website to determine the brand’s visual identity. |
| HTTP Request6 | n8n-nodes-base.httpRequest | Fetches website HTML for logo extraction | On form submission | Information Extractor1 | # 📨 AI Newsletter Automation Workflow<br>## 🚀 Brief Overview (How It Works)<br>This workflow automates the complete process of generating and sending AI-powered newsletters. It starts with a form submission where the user provides a brand name, website, and topic query. The system then fetches relevant news, analyzes the brand’s visual identity (colors & logo), and uses AI to generate structured newsletter topics and content. Finally, it converts everything into a beautifully styled HTML email, saves it to Google Sheets, and sends it to all subscribers via Gmail.<br>## Quick Setup Guide<br>👉 [Demo & Setup Video](https://drive.google.com/file/d/1ulOGWNL3cKe3y7tSKWNG2jlpL7dyiAgX/view?usp=sharing)<br>👉 [Sheet Template](https://docs.google.com/spreadsheets/d/1hvB5Zif52eCLv_X7E_OifQv9OI5usn-CQ50-TsZTMQA/edit?usp=sharing)<br>👉 [Course](https://www.udemy.com/course/n8n-automation-mastery-build-ai-powered-enterprise-ready/?referralCode=2EAE71591D3BEB80F2CC)<br>## Extract Brand Style & Colors<br>This block analyzes the provided brand website to determine the brand’s visual identity. |
| Information Extractor1 | @n8n/n8n-nodes-langchain.informationExtractor | Extracts a logo URL from website HTML | HTTP Request6; OpenAI Chat Model4 | Merge | # 📨 AI Newsletter Automation Workflow<br>## 🚀 Brief Overview (How It Works)<br>This workflow automates the complete process of generating and sending AI-powered newsletters. It starts with a form submission where the user provides a brand name, website, and topic query. The system then fetches relevant news, analyzes the brand’s visual identity (colors & logo), and uses AI to generate structured newsletter topics and content. Finally, it converts everything into a beautifully styled HTML email, saves it to Google Sheets, and sends it to all subscribers via Gmail.<br>## Quick Setup Guide<br>👉 [Demo & Setup Video](https://drive.google.com/file/d/1ulOGWNL3cKe3y7tSKWNG2jlpL7dyiAgX/view?usp=sharing)<br>👉 [Sheet Template](https://docs.google.com/spreadsheets/d/1hvB5Zif52eCLv_X7E_OifQv9OI5usn-CQ50-TsZTMQA/edit?usp=sharing)<br>👉 [Course](https://www.udemy.com/course/n8n-automation-mastery-build-ai-powered-enterprise-ready/?referralCode=2EAE71591D3BEB80F2CC)<br>## Extract Brand Style & Colors<br>This block analyzes the provided brand website to determine the brand’s visual identity. |
| Merge | n8n-nodes-base.merge | Merges logo, brand colors, and content sections | Information Extractor1; AI Agent; Merge Content Pieces | Aggregate | # 📨 AI Newsletter Automation Workflow<br>## 🚀 Brief Overview (How It Works)<br>This workflow automates the complete process of generating and sending AI-powered newsletters. It starts with a form submission where the user provides a brand name, website, and topic query. The system then fetches relevant news, analyzes the brand’s visual identity (colors & logo), and uses AI to generate structured newsletter topics and content. Finally, it converts everything into a beautifully styled HTML email, saves it to Google Sheets, and sends it to all subscribers via Gmail.<br>## Quick Setup Guide<br>👉 [Demo & Setup Video](https://drive.google.com/file/d/1ulOGWNL3cKe3y7tSKWNG2jlpL7dyiAgX/view?usp=sharing)<br>👉 [Sheet Template](https://docs.google.com/spreadsheets/d/1hvB5Zif52eCLv_X7E_OifQv9OI5usn-CQ50-TsZTMQA/edit?usp=sharing)<br>👉 [Course](https://www.udemy.com/course/n8n-automation-mastery-build-ai-powered-enterprise-ready/?referralCode=2EAE71591D3BEB80F2CC)<br>## AI Processing, HTML Generation & Email Delivery<br>This section handles the core content generation and delivery of the newsletter. |
| Aggregate | n8n-nodes-base.aggregate | Consolidates merged items into a single array | Merge | Get row(s) in sheet | # 📨 AI Newsletter Automation Workflow<br>## 🚀 Brief Overview (How It Works)<br>This workflow automates the complete process of generating and sending AI-powered newsletters. It starts with a form submission where the user provides a brand name, website, and topic query. The system then fetches relevant news, analyzes the brand’s visual identity (colors & logo), and uses AI to generate structured newsletter topics and content. Finally, it converts everything into a beautifully styled HTML email, saves it to Google Sheets, and sends it to all subscribers via Gmail.<br>## Quick Setup Guide<br>👉 [Demo & Setup Video](https://drive.google.com/file/d/1ulOGWNL3cKe3y7tSKWNG2jlpL7dyiAgX/view?usp=sharing)<br>👉 [Sheet Template](https://docs.google.com/spreadsheets/d/1hvB5Zif52eCLv_X7E_OifQv9OI5usn-CQ50-TsZTMQA/edit?usp=sharing)<br>👉 [Course](https://www.udemy.com/course/n8n-automation-mastery-build-ai-powered-enterprise-ready/?referralCode=2EAE71591D3BEB80F2CC)<br>## AI Processing, HTML Generation & Email Delivery<br>This section handles the core content generation and delivery of the newsletter. |
| Get row(s) in sheet | n8n-nodes-base.googleSheets | Loads subscriber list from Google Sheets | Aggregate | Loop Over Items | # 📨 AI Newsletter Automation Workflow<br>## 🚀 Brief Overview (How It Works)<br>This workflow automates the complete process of generating and sending AI-powered newsletters. It starts with a form submission where the user provides a brand name, website, and topic query. The system then fetches relevant news, analyzes the brand’s visual identity (colors & logo), and uses AI to generate structured newsletter topics and content. Finally, it converts everything into a beautifully styled HTML email, saves it to Google Sheets, and sends it to all subscribers via Gmail.<br>## Quick Setup Guide<br>👉 [Demo & Setup Video](https://drive.google.com/file/d/1ulOGWNL3cKe3y7tSKWNG2jlpL7dyiAgX/view?usp=sharing)<br>👉 [Sheet Template](https://docs.google.com/spreadsheets/d/1hvB5Zif52eCLv_X7E_OifQv9OI5usn-CQ50-TsZTMQA/edit?usp=sharing)<br>👉 [Course](https://www.udemy.com/course/n8n-automation-mastery-build-ai-powered-enterprise-ready/?referralCode=2EAE71591D3BEB80F2CC)<br>## AI Processing, HTML Generation & Email Delivery<br>This section handles the core content generation and delivery of the newsletter. |
| Loop Over Items | n8n-nodes-base.splitInBatches | Iterates over subscribers | Get row(s) in sheet; Sending Emails to all the Subscribers | Convert Newsletter to HTML (AI) | # 📨 AI Newsletter Automation Workflow<br>## 🚀 Brief Overview (How It Works)<br>This workflow automates the complete process of generating and sending AI-powered newsletters. It starts with a form submission where the user provides a brand name, website, and topic query. The system then fetches relevant news, analyzes the brand’s visual identity (colors & logo), and uses AI to generate structured newsletter topics and content. Finally, it converts everything into a beautifully styled HTML email, saves it to Google Sheets, and sends it to all subscribers via Gmail.<br>## Quick Setup Guide<br>👉 [Demo & Setup Video](https://drive.google.com/file/d/1ulOGWNL3cKe3y7tSKWNG2jlpL7dyiAgX/view?usp=sharing)<br>👉 [Sheet Template](https://docs.google.com/spreadsheets/d/1hvB5Zif52eCLv_X7E_OifQv9OI5usn-CQ50-TsZTMQA/edit?usp=sharing)<br>👉 [Course](https://www.udemy.com/course/n8n-automation-mastery-build-ai-powered-enterprise-ready/?referralCode=2EAE71591D3BEB80F2CC)<br>## AI Processing, HTML Generation & Email Delivery<br>This section handles the core content generation and delivery of the newsletter. |
| Convert Newsletter to HTML (AI) | @n8n/n8n-nodes-langchain.agent | Produces final subject and branded HTML body | Loop Over Items; OpenAI Chat Model2; Structured Output Parser1 | Save Newsletter Draft in Google Sheet; Sending Emails to all the Subscribers | # 📨 AI Newsletter Automation Workflow<br>## 🚀 Brief Overview (How It Works)<br>This workflow automates the complete process of generating and sending AI-powered newsletters. It starts with a form submission where the user provides a brand name, website, and topic query. The system then fetches relevant news, analyzes the brand’s visual identity (colors & logo), and uses AI to generate structured newsletter topics and content. Finally, it converts everything into a beautifully styled HTML email, saves it to Google Sheets, and sends it to all subscribers via Gmail.<br>## Quick Setup Guide<br>👉 [Demo & Setup Video](https://drive.google.com/file/d/1ulOGWNL3cKe3y7tSKWNG2jlpL7dyiAgX/view?usp=sharing)<br>👉 [Sheet Template](https://docs.google.com/spreadsheets/d/1hvB5Zif52eCLv_X7E_OifQv9OI5usn-CQ50-TsZTMQA/edit?usp=sharing)<br>👉 [Course](https://www.udemy.com/course/n8n-automation-mastery-build-ai-powered-enterprise-ready/?referralCode=2EAE71591D3BEB80F2CC)<br>## AI Processing, HTML Generation & Email Delivery<br>This section handles the core content generation and delivery of the newsletter. |
| Structured Output Parser1 | @n8n/n8n-nodes-langchain.outputParserStructured | Structures final subject/content output |  | Convert Newsletter to HTML (AI) | # 📨 AI Newsletter Automation Workflow<br>## 🚀 Brief Overview (How It Works)<br>This workflow automates the complete process of generating and sending AI-powered newsletters. It starts with a form submission where the user provides a brand name, website, and topic query. The system then fetches relevant news, analyzes the brand’s visual identity (colors & logo), and uses AI to generate structured newsletter topics and content. Finally, it converts everything into a beautifully styled HTML email, saves it to Google Sheets, and sends it to all subscribers via Gmail.<br>## Quick Setup Guide<br>👉 [Demo & Setup Video](https://drive.google.com/file/d/1ulOGWNL3cKe3y7tSKWNG2jlpL7dyiAgX/view?usp=sharing)<br>👉 [Sheet Template](https://docs.google.com/spreadsheets/d/1hvB5Zif52eCLv_X7E_OifQv9OI5usn-CQ50-TsZTMQA/edit?usp=sharing)<br>👉 [Course](https://www.udemy.com/course/n8n-automation-mastery-build-ai-powered-enterprise-ready/?referralCode=2EAE71591D3BEB80F2CC)<br>## AI Processing, HTML Generation & Email Delivery<br>This section handles the core content generation and delivery of the newsletter. |
| OpenAI Chat Model2 | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM for final HTML generation |  | Convert Newsletter to HTML (AI) | # 📨 AI Newsletter Automation Workflow<br>## 🚀 Brief Overview (How It Works)<br>This workflow automates the complete process of generating and sending AI-powered newsletters. It starts with a form submission where the user provides a brand name, website, and topic query. The system then fetches relevant news, analyzes the brand’s visual identity (colors & logo), and uses AI to generate structured newsletter topics and content. Finally, it converts everything into a beautifully styled HTML email, saves it to Google Sheets, and sends it to all subscribers via Gmail.<br>## Quick Setup Guide<br>👉 [Demo & Setup Video](https://drive.google.com/file/d/1ulOGWNL3cKe3y7tSKWNG2jlpL7dyiAgX/view?usp=sharing)<br>👉 [Sheet Template](https://docs.google.com/spreadsheets/d/1hvB5Zif52eCLv_X7E_OifQv9OI5usn-CQ50-TsZTMQA/edit?usp=sharing)<br>👉 [Course](https://www.udemy.com/course/n8n-automation-mastery-build-ai-powered-enterprise-ready/?referralCode=2EAE71591D3BEB80F2CC)<br>## AI Processing, HTML Generation & Email Delivery<br>This section handles the core content generation and delivery of the newsletter. |
| Save Newsletter Draft in Google Sheet | n8n-nodes-base.googleSheets | Stores generated newsletter subject/body in a sheet | Convert Newsletter to HTML (AI) |  | # 📨 AI Newsletter Automation Workflow<br>## 🚀 Brief Overview (How It Works)<br>This workflow automates the complete process of generating and sending AI-powered newsletters. It starts with a form submission where the user provides a brand name, website, and topic query. The system then fetches relevant news, analyzes the brand’s visual identity (colors & logo), and uses AI to generate structured newsletter topics and content. Finally, it converts everything into a beautifully styled HTML email, saves it to Google Sheets, and sends it to all subscribers via Gmail.<br>## Quick Setup Guide<br>👉 [Demo & Setup Video](https://drive.google.com/file/d/1ulOGWNL3cKe3y7tSKWNG2jlpL7dyiAgX/view?usp=sharing)<br>👉 [Sheet Template](https://docs.google.com/spreadsheets/d/1hvB5Zif52eCLv_X7E_OifQv9OI5usn-CQ50-TsZTMQA/edit?usp=sharing)<br>👉 [Course](https://www.udemy.com/course/n8n-automation-mastery-build-ai-powered-enterprise-ready/?referralCode=2EAE71591D3BEB80F2CC)<br>## AI Processing, HTML Generation & Email Delivery<br>This section handles the core content generation and delivery of the newsletter. |
| Sending Emails to all the Subscribers | n8n-nodes-base.gmail | Sends final email to each subscriber via Gmail | Convert Newsletter to HTML (AI) | Loop Over Items | # 📨 AI Newsletter Automation Workflow<br>## 🚀 Brief Overview (How It Works)<br>This workflow automates the complete process of generating and sending AI-powered newsletters. It starts with a form submission where the user provides a brand name, website, and topic query. The system then fetches relevant news, analyzes the brand’s visual identity (colors & logo), and uses AI to generate structured newsletter topics and content. Finally, it converts everything into a beautifully styled HTML email, saves it to Google Sheets, and sends it to all subscribers via Gmail.<br>## Quick Setup Guide<br>👉 [Demo & Setup Video](https://drive.google.com/file/d/1ulOGWNL3cKe3y7tSKWNG2jlpL7dyiAgX/view?usp=sharing)<br>👉 [Sheet Template](https://docs.google.com/spreadsheets/d/1hvB5Zif52eCLv_X7E_OifQv9OI5usn-CQ50-TsZTMQA/edit?usp=sharing)<br>👉 [Course](https://www.udemy.com/course/n8n-automation-mastery-build-ai-powered-enterprise-ready/?referralCode=2EAE71591D3BEB80F2CC)<br>## AI Processing, HTML Generation & Email Delivery<br>This section handles the core content generation and delivery of the newsletter. |
| Sticky Note | n8n-nodes-base.stickyNote | Canvas documentation and resource links |  |  |  |
| Sticky Note1 | n8n-nodes-base.stickyNote | Canvas note for input section |  |  |  |
| Sticky Note2 | n8n-nodes-base.stickyNote | Canvas note for brand extraction section |  |  |  |
| Sticky Note3 | n8n-nodes-base.stickyNote | Canvas note for news retrieval section |  |  |  |
| Sticky Note4 | n8n-nodes-base.stickyNote | Canvas note for AI generation and delivery section |  |  |  |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n and name it something like `AI Newsletter Automation Workflow`.

2. **Add a Form Trigger node** named `On form submission`.
   - Set form title to `Newsletter`.
   - Add three required fields:
     1. `Brand Name`
     2. `Brand Website`
     3. `Query`

3. **Add the first Tavily search node** as an `HTTP Request` node named `HTTP Request`.
   - Method: `POST`
   - URL: `https://api.tavily.com/search`
   - Enable headers and JSON body.
   - Header:
     - `Authorization: Bearer YOUR_TAVILY_TOKEN`
   - JSON body:
     - `query: {{ $json.Query }}`
     - `include_answer: advanced`
     - `topic: news`
     - `search_depth: advanced`
     - `time_range: week`
   - Connect `On form submission -> HTTP Request`.

4. **Add an OpenAI Chat Model node** named `OpenAI Chat Model`.
   - Model: `gpt-4.1-mini`
   - Attach valid OpenAI credentials.

5. **Add a Structured Output Parser node** named `Structured Output Parser`.
   - Use manual schema with:
     - `title` as string
     - `topics` as array of exactly 3 strings

6. **Add an AI Agent node** named `Generate Draft Topics (AI)`.
   - Prompt text:
     - Serialize Tavily results from the first search.
   - System message should instruct the model to act as an expert newsletter planner and produce:
     - one short title,
     - exactly three topics of about 5–6 words.
   - Enable structured output.
   - Connect:
     - `HTTP Request -> Generate Draft Topics (AI)`
     - `OpenAI Chat Model -> Generate Draft Topics (AI)` as AI language model
     - `Structured Output Parser -> Generate Draft Topics (AI)` as AI output parser

7. **Add a Split Out node** named `Split Topics`.
   - Field to split out: `output.topics`
   - Connect `Generate Draft Topics (AI) -> Split Topics`.

8. **Add a second Tavily request node** named `Tavily`.
   - Type: `HTTP Request`
   - Method: `POST`
   - URL: `https://api.tavily.com/search`
   - Add Authorization header with Tavily bearer token.
   - Add body parameter for `query`.
   - Use the current topic item as the query.  
     In this exported workflow the expression is `{{ $json['output.topics'] }}`, but when rebuilding you should verify the actual split item structure in execution data. In many cases the split value itself may be under a simpler field path.
   - Connect `Split Topics -> Tavily`.

9. **Add another OpenAI Chat Model node** named `OpenAI Chat Model1`.
   - Model: `gpt-4.1-mini`
   - Use the same OpenAI credential.

10. **Add an AI Agent node** named `Generate Newsletter Content (AI)`.
    - Input text should include:
      - the topic query,
      - the Tavily research results.
    - System message should instruct the model to:
      - write exactly one standalone newsletter section,
      - include a heading,
      - omit overall intro and conclusion,
      - include only real citations with clickable URLs.
    - Connect:
      - `Tavily -> Generate Newsletter Content (AI)`
      - `OpenAI Chat Model1 -> Generate Newsletter Content (AI)`

11. **Add an Aggregate node** named `Merge Content Pieces`.
    - Aggregate the field `output`.
    - Connect `Generate Newsletter Content (AI) -> Merge Content Pieces`.

12. **Add a website fetch node for color extraction** named `HTTP Request2`.
    - Type: `HTTP Request`
    - URL: `={{ $json['Brand Website'] }}`
    - Connect `On form submission -> HTTP Request2`.

13. **Add another website fetch node for logo extraction** named `HTTP Request6`.
    - Type: `HTTP Request`
    - URL: `={{ $json['Brand Website'] }}`
    - Connect `On form submission -> HTTP Request6`.

14. **Add a shared OpenAI Chat Model node** named `OpenAI Chat Model4`.
    - Model: `gpt-4.1-mini`
    - Reuse your OpenAI credentials.

15. **Add a Structured Output Parser node** named `Structured Output Parser2`.
    - Configure a schema/example containing:
      - `primary_color`
      - `secondary_color`
      - `accent_color`
      - `background_color`

16. **Add an AI Agent node** named `AI Agent`.
    - Input text: website HTML from `HTTP Request2`, typically `{{ $json.data }}`
    - System message: analyze the website and extract the color theme using the 4 keys above.
    - Enable structured output.
    - Connect:
      - `HTTP Request2 -> AI Agent`
      - `OpenAI Chat Model4 -> AI Agent`
      - `Structured Output Parser2 -> AI Agent`

17. **Add an Information Extractor node** named `Information Extractor1`.
    - Input text: website HTML from `HTTP Request6`
    - Define one attribute:
      - `Logo` = `Logo Image Link`
    - Connect:
      - `HTTP Request6 -> Information Extractor1`
      - `OpenAI Chat Model4 -> Information Extractor1`

18. **Add a Merge node** named `Merge`.
    - Set number of inputs to `3`.
    - Connect:
      - `Information Extractor1 -> Merge` input 0
      - `AI Agent -> Merge` input 1
      - `Merge Content Pieces -> Merge` input 2

19. **Add an Aggregate node** named `Aggregate`.
    - Set aggregate mode to aggregate all item data.
    - Destination field name: `output`
    - Connect `Merge -> Aggregate`.

20. **Prepare Google Sheets for subscribers and archive data.**
    - Create or reuse a Google spreadsheet.
    - Add:
      - one tab for newsletter archive, e.g. `Sheet1`
      - one tab for subscribers, e.g. `Subscribers List`
    - Subscriber tab should contain at minimum:
      - `Email`
      - optionally `Name`

21. **Add a Google Sheets node** named `Get row(s) in sheet`.
    - Operation: read/get rows
    - Select the spreadsheet and the `Subscribers List` sheet.
    - Connect `Aggregate -> Get row(s) in sheet`.
    - Configure Google Sheets OAuth2 credentials.

22. **Add a Split In Batches node** named `Loop Over Items`.
    - Leave default batch settings unless you want a specific batch size.
    - Connect `Get row(s) in sheet -> Loop Over Items`.

23. **Add an OpenAI Chat Model node** named `OpenAI Chat Model2`.
    - Model: `gpt-4.1-mini`

24. **Add a Structured Output Parser node** named `Structured Output Parser1`.
    - Schema should require:
      - `subject` as string
      - `content` as string

25. **Add an AI Agent node** named `Convert Newsletter to HTML (AI)`.
    - Connect it from the iteration branch of `Loop Over Items`.
    - Configure the prompt to include:
      - newsletter title from `Generate Draft Topics (AI)`
      - three generated sections from `Aggregate`
      - brand colors from `Aggregate`
      - logo URL from `Aggregate`
      - brand name from form submission
      - subscriber name if available
      - current date/year
    - In the exported workflow, the prompt assumes:
      - `Aggregate.output[0]` = logo
      - `Aggregate.output[1]` = colors
      - `Aggregate.output[2]` = section outputs
    - Reproduce the fixed HTML template and styling rules:
      - preserve static layout
      - only text colors should adapt dynamically
      - include intro, section merging, sources, and conclusion
      - return:
        - `subject`
        - `content`
    - Connect:
      - `Loop Over Items -> Convert Newsletter to HTML (AI)`
      - `OpenAI Chat Model2 -> Convert Newsletter to HTML (AI)`
      - `Structured Output Parser1 -> Convert Newsletter to HTML (AI)`

26. **Add a Google Sheets node** named `Save Newsletter Draft in Google Sheet`.
    - Operation: `append`
    - Select the archive sheet, e.g. `Sheet1`
    - Map:
      - `Email` = `{{ $json.output.subject }}`
      - `Message` = `{{ $json.output.content }}`
      - `Email Send At` = `{{$now}}`
      - `Email Created At` = `{{$now}}`
    - Note: the original workflow maps the subject into a column called `Email`. You may want to rename that column to `Subject` when rebuilding for clarity.
    - Connect `Convert Newsletter to HTML (AI) -> Save Newsletter Draft in Google Sheet`.

27. **Add a Gmail node** named `Sending Emails to all the Subscribers`.
    - Operation: send email
    - To:
      - `={{ $('Loop Over Items').item.json.Email }}`
    - Subject:
      - `={{ $('Convert Newsletter to HTML (AI)').item.json.output.subject }}`
    - Message:
      - `={{ $('Convert Newsletter to HTML (AI)').item.json.output.content }}`
    - Configure Gmail OAuth2 credentials.
    - Connect `Convert Newsletter to HTML (AI) -> Sending Emails to all the Subscribers`.

28. **Close the loop** by connecting:
    - `Sending Emails to all the Subscribers -> Loop Over Items`
    This allows the workflow to continue to the next subscriber.

29. **Add optional sticky notes** to document the canvas:
    - Overview and setup resources
    - Input section
    - Brand extraction section
    - News retrieval section
    - AI processing and delivery section

30. **Test with a controlled dataset.**
    - Use a small subscriber list first.
    - Verify:
      - Tavily token works,
      - OpenAI output matches parser schemas,
      - logo URL is valid,
      - color strings are usable CSS values,
      - Gmail sends HTML as intended,
      - aggregate array indexing matches your rebuilt flow.

31. **Recommended hardening after rebuild:**
    - Add error handling for failed website fetches.
    - Normalize logo URLs to absolute URLs.
    - Add an IF node to skip empty subscriber emails.
    - Add retry or fallback when Tavily returns no results.
    - Replace duplicate website fetches with one fetch plus branching, if desired.
    - Consider generating the newsletter once, then sending the same result to all subscribers instead of recomputing final HTML per recipient unless personalization is needed.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Demo & Setup Video | https://drive.google.com/file/d/1ulOGWNL3cKe3y7tSKWNG2jlpL7dyiAgX/view?usp=sharing |
| Sheet Template | https://docs.google.com/spreadsheets/d/1hvB5Zif52eCLv_X7E_OifQv9OI5usn-CQ50-TsZTMQA/edit?usp=sharing |
| Course | https://www.udemy.com/course/n8n-automation-mastery-build-ai-powered-enterprise-ready/?referralCode=2EAE71591D3BEB80F2CC |
| Workflow purpose note | Automates AI-powered newsletter creation, branding, storage, and delivery from form submission to Gmail sending. |
| Important implementation note | The final HTML generation depends on positional array references from the `Aggregate` node. If merge order changes, the prompt expressions must be updated. |
| Important implementation note | The workflow has multiple external dependencies: Tavily API, OpenAI API, Google Sheets OAuth2, Gmail OAuth2, and public accessibility of the submitted brand website. |

If you want, I can also convert this into a more machine-friendly spec format, such as:
- a node dependency graph,
- a condensed implementation manifest,
- or a QA checklist for validating the workflow after import.