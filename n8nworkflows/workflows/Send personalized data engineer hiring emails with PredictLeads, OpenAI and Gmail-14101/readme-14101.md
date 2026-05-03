Send personalized data engineer hiring emails with PredictLeads, OpenAI and Gmail

https://n8nworkflows.xyz/workflows/send-personalized-data-engineer-hiring-emails-with-predictleads--openai-and-gmail-14101


# Send personalized data engineer hiring emails with PredictLeads, OpenAI and Gmail

# 1. Workflow Overview

This workflow identifies companies that appear to be hiring for **Data Engineer** roles, enriches those companies with **PredictLeads** company and technology data, uses **OpenAI** to generate a tailored outreach message, and sends the message through **Gmail**.

Its main use case is outbound prospecting for a data consultancy or services firm: the workflow looks for hiring signals that suggest a company may need help with data pipelines, warehousing, analytics infrastructure, or related engineering work.

## 1.1 Trigger & Input Initialization

The workflow starts on a daily schedule, then sets a target domain manually in a Set node. That domain is used as the initial company identifier for PredictLeads API requests.

## 1.2 Job Discovery

The workflow queries PredictLeads for job openings on the target domain, filtered to the title **“data engineer”**. This is the hiring signal used to qualify the company.

## 1.3 Per-Company Enrichment Loop

The job opening results are passed into a **Split in Batches** loop. For each iteration, the workflow fetches company profile data and technology detection data from PredictLeads.

## 1.4 AI Personalization

The workflow builds a custom prompt from the available company context and sends it to OpenAI’s Chat Completions API to generate outreach copy.

## 1.5 Email Delivery & Loop Continuation

The generated email body is sent through Gmail. After sending, the workflow loops back into the batch node to continue processing remaining items.

---

# 2. Block-by-Block Analysis

## 2.1 Informational / Documentation Layer

### Overview
This block contains sticky notes that explain the workflow purpose and visually group the main phases. These nodes do not affect execution, but they are important for maintainability and operator understanding.

### Nodes Involved
- About This Workflow
- 📋 Trigger & Discovery
- 🔬 Enrichment
- 🤖 AI Personalization

### Node Details

#### About This Workflow
- **Type / role:** Sticky Note; visual documentation node.
- **Configuration choices:** Contains a summary of the workflow purpose, setup requirements, use case, PredictLeads link, and contact link.
- **Key expressions / variables used:** None.
- **Input / output connections:** None.
- **Version-specific requirements:** Sticky note typeVersion 1.
- **Edge cases / failures:** None at runtime.
- **Sub-workflow reference:** None.

#### 📋 Trigger & Discovery
- **Type / role:** Sticky Note; labels the trigger and discovery section.
- **Configuration choices:** Describes the schedule, domain setting, and PredictLeads job-opening fetch.
- **Key expressions / variables used:** None.
- **Input / output connections:** None.
- **Version-specific requirements:** Sticky note typeVersion 1.
- **Edge cases / failures:** None at runtime.
- **Sub-workflow reference:** None.

#### 🔬 Enrichment
- **Type / role:** Sticky Note; labels the enrichment section.
- **Configuration choices:** Documents the split loop and PredictLeads enrichment calls.
- **Key expressions / variables used:** None.
- **Input / output connections:** None.
- **Version-specific requirements:** Sticky note typeVersion 1.
- **Edge cases / failures:** None at runtime.
- **Sub-workflow reference:** None.

#### 🤖 AI Personalization
- **Type / role:** Sticky Note; labels the prompt building, AI generation, and email sending section.
- **Configuration choices:** Describes prompt creation, AI personalization, and delivery via Gmail.
- **Key expressions / variables used:** None.
- **Input / output connections:** None.
- **Version-specific requirements:** Sticky note typeVersion 1.
- **Edge cases / failures:** None at runtime.
- **Sub-workflow reference:** None.

---

## 2.2 Trigger & Domain Setup

### Overview
This block starts the workflow every day and defines the target domain used throughout the rest of the process. The domain is currently hardcoded, making this more of a single-company daily outreach pattern than a broad multi-company discovery workflow.

### Nodes Involved
- ⏰ Daily Schedule
- set domain

### Node Details

#### ⏰ Daily Schedule
- **Type / role:** `n8n-nodes-base.scheduleTrigger`; entry point.
- **Configuration choices:** Configured to trigger on an interval rule with `triggerAtHour: 8`, meaning the workflow runs daily at 08:00 according to the n8n instance timezone.
- **Key expressions / variables used:** None.
- **Input / output connections:** No input; outputs to **set domain**.
- **Version-specific requirements:** typeVersion 1.2.
- **Edge cases / failures:**
  - Timezone differences between expected local time and server time.
  - Workflow must be activated for the trigger to run.
- **Sub-workflow reference:** None.

#### set domain
- **Type / role:** `n8n-nodes-base.set`; initializes the domain variable.
- **Configuration choices:** Assigns a string field named `domain` with the value `stripe.com`.
- **Key expressions / variables used:** None; static assignment.
- **Input / output connections:** Input from **⏰ Daily Schedule**; output to **🔍 Fetch Data Engineer Job Openings**.
- **Version-specific requirements:** typeVersion 3.4.
- **Edge cases / failures:**
  - If changed to an invalid domain, downstream API calls may return 404/empty results.
  - Because only one domain is set, the workflow does not truly discover multiple companies unless manually adapted.
- **Sub-workflow reference:** None.

---

## 2.3 Job Discovery

### Overview
This block queries PredictLeads for job openings for the specified company domain, filtering on the title “data engineer”. The results are the basis for deciding whether the company is currently investing in data engineering capabilities.

### Nodes Involved
- 🔍 Fetch Data Engineer Job Openings

### Node Details

#### 🔍 Fetch Data Engineer Job Openings
- **Type / role:** `n8n-nodes-base.httpRequest`; calls PredictLeads job openings endpoint.
- **Configuration choices:**
  - URL is dynamic:  
    `https://predictleads.com/api/v3/companies/{{ $json.domain }}/job_openings`
  - Query parameter `title=data engineer`
  - Headers:
    - `X-Api-Key`
    - `X-Api-Token`
    - `Content-Type: application/json`
- **Key expressions / variables used:**
  - `{{ $json.domain }}` from the previous Set node.
- **Input / output connections:** Input from **set domain**; output to **🔀 Split by Company**.
- **Version-specific requirements:** typeVersion 4.2.
- **Edge cases / failures:**
  - Invalid or expired PredictLeads credentials.
  - No matching job openings returned.
  - API rate limits or temporary service unavailability.
  - Unexpected response structure that may affect downstream batching.
- **Sub-workflow reference:** None.

**Important implementation note:**  
Although the sticky note suggests discovery of companies hiring data engineers, the actual request is scoped to a **single preset domain**. This means the node checks whether *that one company* has matching job openings; it does not search the broader market.

---

## 2.4 Per-Company Enrichment Loop

### Overview
This block iterates through the fetched items and enriches each one with company profile and technology data from PredictLeads. The loop is implemented with a Split in Batches node and a feedback connection from the Gmail node.

### Nodes Involved
- 🔀 Split by Company
- 🏢 Get Company Info
- 💻 Get Technology Stack

### Node Details

#### 🔀 Split by Company
- **Type / role:** `n8n-nodes-base.splitInBatches`; processes items one at a time in a loop.
- **Configuration choices:** No custom batch size options are explicitly set; default behavior applies.
- **Key expressions / variables used:** None.
- **Input / output connections:**
  - Input from **🔍 Fetch Data Engineer Job Openings**
  - Loop continuation input from **📧 Send Personalized Email**
  - Second output goes to **🏢 Get Company Info**
  - First output is the completion path and is unused
- **Version-specific requirements:** typeVersion 3.
- **Edge cases / failures:**
  - If the incoming job-openings response is not itemized as expected, batching may not behave as intended.
  - Empty input means the enrichment path never runs.
  - Misconfigured loop-back can cause unexpected execution patterns if additional branches are added later.
- **Sub-workflow reference:** None.

#### 🏢 Get Company Info
- **Type / role:** `n8n-nodes-base.httpRequest`; fetches company-level metadata from PredictLeads.
- **Configuration choices:**
  - URL:
    `https://predictleads.com/api/v3/companies/{{ $('set domain').item.json.domain }}`
  - Headers:
    - `X-Api-Key`
    - `X-Api-Token`
    - `Content-Type: application/json`
- **Key expressions / variables used:**
  - `{{ $('set domain').item.json.domain }}`
- **Input / output connections:** Input from **🔀 Split by Company**; output to **💻 Get Technology Stack**.
- **Version-specific requirements:** typeVersion 4.2.
- **Edge cases / failures:**
  - PredictLeads auth failures.
  - Domain not found in PredictLeads.
  - Expression lookup failure if the Set node name changes.
- **Sub-workflow reference:** None.

**Important design note:**  
This node does **not** use the current batch item to determine the company. It always references the `set domain` node, so every loop iteration fetches the same company info.

#### 💻 Get Technology Stack
- **Type / role:** `n8n-nodes-base.httpRequest`; fetches detected technologies for the company from PredictLeads.
- **Configuration choices:**
  - URL:
    `https://predictleads.com/api/v3/companies/{{ $('set domain').item.json.domain }}/technology_detections`
  - Headers:
    - `X-Api-Key`
    - `X-Api-Token`
    - `Content-Type: application/json`
- **Key expressions / variables used:**
  - `{{ $('set domain').item.json.domain }}`
- **Input / output connections:** Input from **🏢 Get Company Info**; output to **⚙️ Build Personalized Prompt**.
- **Version-specific requirements:** typeVersion 4.2.
- **Edge cases / failures:**
  - PredictLeads auth errors or rate limits.
  - Technology detections may be empty.
  - Because the node consumes the previous node’s item stream, data shape may not match prompt-builder expectations.
- **Sub-workflow reference:** None.

**Important design note:**  
Like the company info node, this also always uses the fixed domain from **set domain**, not a per-item company identifier.

---

## 2.5 AI Prompt Construction

### Overview
This block transforms the enrichment data into a structured prompt for OpenAI. It attempts to combine company profile fields and technology data into a sales outreach request.

### Nodes Involved
- ⚙️ Build Personalized Prompt

### Node Details

#### ⚙️ Build Personalized Prompt
- **Type / role:** `n8n-nodes-base.code`; custom JavaScript transformation.
- **Configuration choices:**
  - Reads all incoming items with `$input.all()`
  - Pulls `domain` from `$('set domain').first().json.domain`
  - Derives company name from the domain by stripping the TLD
  - Tries to map company details from `item.json.data`
  - Builds a prompt asking OpenAI to generate a personalized B2B outreach email
  - Returns items with:
    - `company`
    - `domain`
    - `industry`
    - `tech_stack`
    - `prompt`
- **Key expressions / variables used:**
  - `$('set domain').first().json.domain`
  - `item.json.data`
  - JavaScript string assembly for the prompt
- **Input / output connections:** Input from **💻 Get Technology Stack**; output to **🤖 Generate Personalized Email**.
- **Version-specific requirements:** typeVersion 2.
- **Edge cases / failures:**
  - The code assumes `item.json.data` can represent both a company object and a technology array, which is structurally inconsistent.
  - If PredictLeads technology response does not expose an array at `item.json.data`, `.map()` may fail.
  - If company description or industry fields are absent, defaults are used.
  - If node names change, expression references to `set domain` will break.
- **Sub-workflow reference:** None.

**Important logic issue:**  
The node receives input directly from **💻 Get Technology Stack**, so it only has that node’s output unless prior data was merged elsewhere. However, the code expects both company profile data and technology data to be available in the same `item.json.data`. Since there is no Merge node, the prompt is likely built from incomplete or mismatched data.

---

## 2.6 AI Email Generation

### Overview
This block sends the generated prompt to OpenAI’s Chat Completions API. The API is asked to produce a concise, personalized outreach email for data engineering services.

### Nodes Involved
- 🤖 Generate Personalized Email

### Node Details

#### 🤖 Generate Personalized Email
- **Type / role:** `n8n-nodes-base.httpRequest`; direct OpenAI API call.
- **Configuration choices:**
  - Method: `POST`
  - URL: `https://api.openai.com/v1/chat/completions`
  - JSON body includes:
    - `model: gpt-4o-mini`
    - system message defining the role
    - user message set from `$json.prompt`
    - `temperature: 0.7`
    - `max_tokens: 500`
  - Headers:
    - `Authorization: Bearer YOUR_TOKEN_HERE`
    - `Content-Type: application/json`
- **Key expressions / variables used:**
  - `{{ JSON.stringify($json.prompt) }}`
- **Input / output connections:** Input from **⚙️ Build Personalized Prompt**; output to **📧 Send Personalized Email**.
- **Version-specific requirements:** typeVersion 4.2.
- **Edge cases / failures:**
  - Invalid OpenAI API token.
  - Model availability or account access restrictions.
  - Quota exhaustion or rate limiting.
  - Prompt missing or malformed.
  - Response format changes could break downstream `choices[0].message.content` access.
- **Sub-workflow reference:** None.

**Implementation note:**  
This uses a generic HTTP Request node rather than the dedicated OpenAI node, so credential storage and error handling are more manual.

---

## 2.7 Email Delivery & Loop Control

### Overview
This block sends the generated email via Gmail and then returns control to the batch node to process the next item. It is the final action step and also part of the batching mechanism.

### Nodes Involved
- 📧 Send Personalized Email

### Node Details

#### 📧 Send Personalized Email
- **Type / role:** `n8n-nodes-base.gmail`; sends an email through Gmail OAuth2.
- **Configuration choices:**
  - Recipient:
    `{{ $('⚙️ Build Personalized Prompt').item.json.domain.replace(/\..*/, '') }}@example.com`
  - Subject:
    `Data Engineering Services for {{ $('⚙️ Build Personalized Prompt').item.json.company }}`
  - Message body:
    `{{ $json.choices[0].message.content }}`
  - Email type: plain text
  - Uses configured Gmail OAuth2 credential named **Gmail account**
- **Key expressions / variables used:**
  - `$('⚙️ Build Personalized Prompt').item.json.domain.replace(/\..*/, '')`
  - `$('⚙️ Build Personalized Prompt').item.json.company`
  - `$json.choices[0].message.content`
- **Input / output connections:**
  - Input from **🤖 Generate Personalized Email**
  - Output back to **🔀 Split by Company** to continue the loop
- **Version-specific requirements:** typeVersion 2.1.
- **Edge cases / failures:**
  - Gmail OAuth2 credential expiry or missing scopes.
  - Recipient address is synthetic and likely invalid in real use because it becomes values like `stripe@example.com`.
  - If OpenAI response lacks `choices[0].message.content`, email body generation fails.
  - Gmail sending limits may apply.
- **Sub-workflow reference:** None.

**Important design note:**  
The subject line is static apart from company name. Although the prompt asks OpenAI to include a subject line, the workflow does not parse or use one. The generated content is inserted only into the body.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| About This Workflow | Sticky Note | General workflow documentation |  |  | ABOUT THIS WORKFLOW\nFinds companies hiring data engineers, enriches them with tech stack data from PredictLeads, and sends AI-personalized service offering emails.\nSetup: PredictLeads API, OpenAI API key, Gmail OAuth2.\nUse case: A data consultancy finds companies that posted data engineer roles, then sends them a tailored pitch referencing their current stack (e.g., "I see you're on Snowflake and looking for pipeline engineers").\nPredictLeads API: https://predictleads.com\nQuestions: https://www.linkedin.com/in/yaronbeen |
| 📋 Trigger & Discovery | Sticky Note | Visual section label for trigger and discovery |  |  | ## ⏰ Trigger & Job Discovery\n**Nodes:**\n⏰ Daily Schedule → set domain → 🔍 Fetch Data Engineer Job Openings\n**Description:**\nThe workflow starts on a daily schedule that automatically triggers the process.\nA domain is defined and used to query the PredictLeads API for **job openings related to “Data Engineer” roles**.\nThis step identifies companies that are actively investing in data infrastructure or analytics capabilities.\nThese hiring signals act as strong indicators that the company may require external **data engineering services or support**. |
| 🔬 Enrichment | Sticky Note | Visual section label for enrichment |  |  | ## 🔍 Company Enrichment\n**Nodes:**\n🔀 Split by Company → 🏢 Get Company Info → 💻 Get Technology Stack\n**Description:**\nEach company identified in the job discovery step is processed individually.\nThe workflow enriches the company using PredictLeads data sources to gather additional context including:\n- Company profile information\n- Industry and company size\n- Current technology stack\nThis enrichment step helps build a deeper understanding of the company’s technical environment and hiring needs, enabling more relevant outreach. |
| 🤖 AI Personalization | Sticky Note | Visual section label for prompting, generation, and email delivery |  |  | ## 🤖 AI Personalization & Outreach\n**Nodes:**\n⚙️ Build Personalized Prompt → 🤖 Generate Personalized Email → 📧 Send Personalized Email\n**Description:**\nA detailed prompt is created by combining company information, industry context, and the detected technology stack.\nOpenAI then generates a **personalized outreach email** tailored to the company’s hiring activity and current technical environment.\nThe message references their tools (for example Snowflake or other data platforms) and proposes relevant services such as data pipeline development, data warehouse optimization, or analytics infrastructure support.\nFinally, the AI-generated email is automatically sent via Gmail with a personalized subject line, delivering targeted outreach to companies actively hiring data engineers. |
| ⏰ Daily Schedule | Schedule Trigger | Daily workflow trigger at 08:00 |  | set domain | ## ⏰ Trigger & Job Discovery\n**Nodes:**\n⏰ Daily Schedule → set domain → 🔍 Fetch Data Engineer Job Openings\n**Description:**\nThe workflow starts on a daily schedule that automatically triggers the process.\nA domain is defined and used to query the PredictLeads API for **job openings related to “Data Engineer” roles**.\nThis step identifies companies that are actively investing in data infrastructure or analytics capabilities.\nThese hiring signals act as strong indicators that the company may require external **data engineering services or support**. |
| set domain | Set | Defines the target company domain | ⏰ Daily Schedule | 🔍 Fetch Data Engineer Job Openings | ## ⏰ Trigger & Job Discovery\n**Nodes:**\n⏰ Daily Schedule → set domain → 🔍 Fetch Data Engineer Job Openings\n**Description:**\nThe workflow starts on a daily schedule that automatically triggers the process.\nA domain is defined and used to query the PredictLeads API for **job openings related to “Data Engineer” roles**.\nThis step identifies companies that are actively investing in data infrastructure or analytics capabilities.\nThese hiring signals act as strong indicators that the company may require external **data engineering services or support**. |
| 🔍 Fetch Data Engineer Job Openings | HTTP Request | Queries PredictLeads job openings for the configured domain | set domain | 🔀 Split by Company | ## ⏰ Trigger & Job Discovery\n**Nodes:**\n⏰ Daily Schedule → set domain → 🔍 Fetch Data Engineer Job Openings\n**Description:**\nThe workflow starts on a daily schedule that automatically triggers the process.\nA domain is defined and used to query the PredictLeads API for **job openings related to “Data Engineer” roles**.\nThis step identifies companies that are actively investing in data infrastructure or analytics capabilities.\nThese hiring signals act as strong indicators that the company may require external **data engineering services or support**. |
| 🔀 Split by Company | Split In Batches | Iterates through discovered items in a loop | 🔍 Fetch Data Engineer Job Openings; 📧 Send Personalized Email | 🏢 Get Company Info | ## 🔍 Company Enrichment\n**Nodes:**\n🔀 Split by Company → 🏢 Get Company Info → 💻 Get Technology Stack\n**Description:**\nEach company identified in the job discovery step is processed individually.\nThe workflow enriches the company using PredictLeads data sources to gather additional context including:\n- Company profile information\n- Industry and company size\n- Current technology stack\nThis enrichment step helps build a deeper understanding of the company’s technical environment and hiring needs, enabling more relevant outreach. |
| 🏢 Get Company Info | HTTP Request | Fetches company metadata from PredictLeads | 🔀 Split by Company | 💻 Get Technology Stack | ## 🔍 Company Enrichment\n**Nodes:**\n🔀 Split by Company → 🏢 Get Company Info → 💻 Get Technology Stack\n**Description:**\nEach company identified in the job discovery step is processed individually.\nThe workflow enriches the company using PredictLeads data sources to gather additional context including:\n- Company profile information\n- Industry and company size\n- Current technology stack\nThis enrichment step helps build a deeper understanding of the company’s technical environment and hiring needs, enabling more relevant outreach. |
| 💻 Get Technology Stack | HTTP Request | Fetches technology detections from PredictLeads | 🏢 Get Company Info | ⚙️ Build Personalized Prompt | ## 🔍 Company Enrichment\n**Nodes:**\n🔀 Split by Company → 🏢 Get Company Info → 💻 Get Technology Stack\n**Description:**\nEach company identified in the job discovery step is processed individually.\nThe workflow enriches the company using PredictLeads data sources to gather additional context including:\n- Company profile information\n- Industry and company size\n- Current technology stack\nThis enrichment step helps build a deeper understanding of the company’s technical environment and hiring needs, enabling more relevant outreach. |
| ⚙️ Build Personalized Prompt | Code | Constructs the AI prompt from company and tech data | 💻 Get Technology Stack | 🤖 Generate Personalized Email | ## 🤖 AI Personalization & Outreach\n**Nodes:**\n⚙️ Build Personalized Prompt → 🤖 Generate Personalized Email → 📧 Send Personalized Email\n**Description:**\nA detailed prompt is created by combining company information, industry context, and the detected technology stack.\nOpenAI then generates a **personalized outreach email** tailored to the company’s hiring activity and current technical environment.\nThe message references their tools (for example Snowflake or other data platforms) and proposes relevant services such as data pipeline development, data warehouse optimization, or analytics infrastructure support.\nFinally, the AI-generated email is automatically sent via Gmail with a personalized subject line, delivering targeted outreach to companies actively hiring data engineers. |
| 🤖 Generate Personalized Email | HTTP Request | Calls OpenAI Chat Completions to generate email copy | ⚙️ Build Personalized Prompt | 📧 Send Personalized Email | ## 🤖 AI Personalization & Outreach\n**Nodes:**\n⚙️ Build Personalized Prompt → 🤖 Generate Personalized Email → 📧 Send Personalized Email\n**Description:**\nA detailed prompt is created by combining company information, industry context, and the detected technology stack.\nOpenAI then generates a **personalized outreach email** tailored to the company’s hiring activity and current technical environment.\nThe message references their tools (for example Snowflake or other data platforms) and proposes relevant services such as data pipeline development, data warehouse optimization, or analytics infrastructure support.\nFinally, the AI-generated email is automatically sent via Gmail with a personalized subject line, delivering targeted outreach to companies actively hiring data engineers. |
| 📧 Send Personalized Email | Gmail | Sends the generated outreach email and advances the loop | 🤖 Generate Personalized Email | 🔀 Split by Company | ## 🤖 AI Personalization & Outreach\n**Nodes:**\n⚙️ Build Personalized Prompt → 🤖 Generate Personalized Email → 📧 Send Personalized Email\n**Description:**\nA detailed prompt is created by combining company information, industry context, and the detected technology stack.\nOpenAI then generates a **personalized outreach email** tailored to the company’s hiring activity and current technical environment.\nThe message references their tools (for example Snowflake or other data platforms) and proposes relevant services such as data pipeline development, data warehouse optimization, or analytics infrastructure support.\nFinally, the AI-generated email is automatically sent via Gmail with a personalized subject line, delivering targeted outreach to companies actively hiring data engineers. |

---

# 4. Reproducing the Workflow from Scratch

Below is the exact build order to recreate the workflow in n8n.

## 1. Create the schedule trigger
1. Add a **Schedule Trigger** node.
2. Name it **⏰ Daily Schedule**.
3. Set it to run **daily at 08:00**.
4. Confirm the timezone used by your n8n instance.

## 2. Create the domain initialization node
5. Add a **Set** node.
6. Name it **set domain**.
7. Add one field:
   - **domain** = `stripe.com`
8. Connect **⏰ Daily Schedule → set domain**.

## 3. Create the PredictLeads job openings request
9. Add an **HTTP Request** node.
10. Name it **🔍 Fetch Data Engineer Job Openings**.
11. Set:
   - **Method:** GET
   - **URL:** `=https://predictleads.com/api/v3/companies/{{ $json.domain }}/job_openings`
12. Enable query parameters and add:
   - `title` = `data engineer`
13. Enable headers and add:
   - `X-Api-Key` = your PredictLeads API key
   - `X-Api-Token` = your PredictLeads API token
   - `Content-Type` = `application/json`
14. Connect **set domain → 🔍 Fetch Data Engineer Job Openings**.

## 4. Create the batching loop
15. Add a **Split In Batches** node.
16. Name it **🔀 Split by Company**.
17. Leave default batch settings unless you want to explicitly set batch size to 1.
18. Connect **🔍 Fetch Data Engineer Job Openings → 🔀 Split by Company**.

## 5. Create the company enrichment request
19. Add an **HTTP Request** node.
20. Name it **🏢 Get Company Info**.
21. Set:
   - **Method:** GET
   - **URL:** `=https://predictleads.com/api/v3/companies/{{ $('set domain').item.json.domain }}`
22. Add headers:
   - `X-Api-Key` = your PredictLeads API key
   - `X-Api-Token` = your PredictLeads API token
   - `Content-Type` = `application/json`
23. Connect the **second/main loop output** of **🔀 Split by Company** to **🏢 Get Company Info**.

## 6. Create the technology stack request
24. Add another **HTTP Request** node.
25. Name it **💻 Get Technology Stack**.
26. Set:
   - **Method:** GET
   - **URL:** `=https://predictleads.com/api/v3/companies/{{ $('set domain').item.json.domain }}/technology_detections`
27. Add headers:
   - `X-Api-Key` = your PredictLeads API key
   - `X-Api-Token` = your PredictLeads API token
   - `Content-Type` = `application/json`
28. Connect **🏢 Get Company Info → 💻 Get Technology Stack**.

## 7. Create the prompt builder
29. Add a **Code** node.
30. Name it **⚙️ Build Personalized Prompt**.
31. Paste this JavaScript logic equivalent:

   - Read all input items.
   - Read the domain from the **set domain** node.
   - Derive company name from the domain by removing the TLD.
   - Try to read company fields such as industry, employee_count, and description.
   - Read technology names and join them into a comma-separated list.
   - Build a prompt asking OpenAI to draft a professional but friendly personalized outreach email.
   - Return a JSON object containing `company`, `domain`, `industry`, `tech_stack`, and `prompt`.

32. Connect **💻 Get Technology Stack → ⚙️ Build Personalized Prompt**.

### Code used in the original workflow
Use this code if you want to match the original behavior exactly:

```javascript
const items = $input.all();
const results = [];

// domain from Set Domain node
const domain = $('set domain').first().json.domain || '';

for (const item of items) {

  const companyData = item.json.data || {};

  // company name from domain (remove .com or any TLD)
  const companyName = domain.split('.')[0];

  const industry =
    companyData.industry ||
    'technology';

  const size =
    companyData.employee_count ||
    'unknown';

  const description =
    companyData.description ||
    '';

  const technologies = item.json.data || [];

  const techList = technologies
    .map(t => t.name || t.technology_name)
    .filter(Boolean)
    .join(', ');

  const cleanTech =
    techList.length ? techList : 'not available';

  const prompt = `You are a B2B sales expert specializing in data engineering services. Draft a personalized outreach email.

Context:
Company: ${companyName}
Domain: ${domain}
Industry: ${industry}
Company size: ${size}
Company description: ${description}
Their current tech stack includes: ${cleanTech}

Requirements:
- Subject line included
- Mention their tech stack and how your services add value
- Offer solutions like data pipelines, warehouse optimization etc.
- Professional but friendly tone
- Clear call-to-action for a short intro call`;

  results.push({
    json: {
      company: companyName,
      domain: domain,
      industry: industry,
      tech_stack: cleanTech,
      prompt: prompt
    }
  });

}

return results;
```

## 8. Create the OpenAI API call
33. Add an **HTTP Request** node.
34. Name it **🤖 Generate Personalized Email**.
35. Set:
   - **Method:** POST
   - **URL:** `https://api.openai.com/v1/chat/completions`
36. Set body type to **JSON**.
37. Enable headers and add:
   - `Authorization` = `Bearer YOUR_TOKEN_HERE`
   - `Content-Type` = `application/json`
38. Set the JSON body to:

```json
{
  "model": "gpt-4o-mini",
  "messages": [
    {
      "role": "system",
      "content": "You are a B2B sales expert specializing in data engineering services. Write concise, personalized outreach emails that demonstrate deep understanding of the prospect's needs."
    },
    {
      "role": "user",
      "content": {{ JSON.stringify($json.prompt) }}
    }
  ],
  "temperature": 0.7,
  "max_tokens": 500
}
```

39. Connect **⚙️ Build Personalized Prompt → 🤖 Generate Personalized Email**.

## 9. Create the Gmail sending node
40. Add a **Gmail** node.
41. Name it **📧 Send Personalized Email**.
42. Choose the operation for sending email.
43. Configure Gmail OAuth2 credentials:
   - Authenticate the Gmail account to send from.
   - Ensure send-mail scopes are granted.
44. Set:
   - **To:** `={{ $('⚙️ Build Personalized Prompt').item.json.domain.replace(/\..*/, '') }}@example.com`
   - **Subject:** `=Data Engineering Services for {{ $('⚙️ Build Personalized Prompt').item.json.company }}`
   - **Message:** `={{ $json.choices[0].message.content }}`
   - **Email Type:** Text
45. Connect **🤖 Generate Personalized Email → 📧 Send Personalized Email**.

## 10. Close the loop
46. Connect **📧 Send Personalized Email → 🔀 Split by Company**.
47. Use the loop connection so the batch node continues to the next item after each send.

## 11. Add the sticky notes
48. Add a **Sticky Note** named **About This Workflow** and paste the descriptive text from the original workflow.
49. Add a **Sticky Note** named **📋 Trigger & Discovery** above the trigger/discovery nodes.
50. Add a **Sticky Note** named **🔬 Enrichment** above the split and PredictLeads enrichment nodes.
51. Add a **Sticky Note** named **🤖 AI Personalization** above the Code, OpenAI, and Gmail nodes.

## 12. Activate and test
52. Save the workflow.
53. Run a manual test first.
54. Verify:
   - PredictLeads returns job opening items.
   - Split In Batches iterates correctly.
   - OpenAI returns `choices[0].message.content`.
   - Gmail successfully sends.
55. Activate the workflow only after credential and response-shape validation.

---

## Credential configuration details

### PredictLeads
This workflow uses raw header authentication in HTTP Request nodes rather than stored n8n credentials.
You must provide:
- `X-Api-Key`
- `X-Api-Token`

Recommended improvement: convert these to environment variables or n8n credentials-backed expressions rather than hardcoding.

### OpenAI
The OpenAI token is inserted manually in the Authorization header:
- `Authorization: Bearer YOUR_TOKEN_HERE`

Recommended improvement: use a dedicated credential or environment variable.

### Gmail OAuth2
You need a Gmail OAuth2 credential in n8n with permission to send mail. The original workflow references:
- Credential name: **Gmail account**

---

## Practical constraints and implementation caveats

1. **The workflow is not truly discovering new company domains.**  
   It only checks one hardcoded domain from **set domain**.

2. **The loop is somewhat misleading.**  
   Multiple job-opening items may be processed, but every enrichment call still uses the same domain from **set domain**.

3. **Company data and tech data are not merged.**  
   The Code node expects combined context, but the workflow does not include a Merge node.

4. **The recipient email is placeholder logic.**  
   `stripe@example.com`-style addresses are not real unless intentionally used for testing.

5. **The AI-generated subject line is unused.**  
   The prompt requests a subject line, but Gmail uses a separate static subject field.

6. **Response-shape validation is essential.**  
   If PredictLeads returns arrays, nested `data`, or alternate field names, the Code node may fail.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| PredictLeads API | https://predictleads.com |
| Questions / creator contact | https://www.linkedin.com/in/yaronbeen |
| Setup requirements mentioned in the workflow | PredictLeads API, OpenAI API key, Gmail OAuth2 |
| Workflow purpose | Find companies hiring data engineers, enrich with tech stack data, and send AI-personalized service offering emails |
| Example positioning from the note | A data consultancy can reference the company’s current stack, such as Snowflake, and pitch pipeline or warehouse-related services |