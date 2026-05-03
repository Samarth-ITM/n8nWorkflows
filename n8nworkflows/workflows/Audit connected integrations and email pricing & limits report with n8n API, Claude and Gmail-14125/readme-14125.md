Audit connected integrations and email pricing & limits report with n8n API, Claude and Gmail

https://n8nworkflows.xyz/workflows/audit-connected-integrations-and-email-pricing---limits-report-with-n8n-api--claude-and-gmail-14125


# Audit connected integrations and email pricing & limits report with n8n API, Claude and Gmail

# 1. Workflow Overview

This workflow audits the integrations currently connected to an n8n instance by reading the instance’s stored credentials through the n8n API, translating credential types into recognizable service names, asking Claude to research pricing/free-tier/rate-limit information for those services, and sending a formatted HTML summary by Gmail.

Typical use cases:

- Internal review of which external services are connected to an n8n workspace
- Cost-awareness and governance reporting
- Periodic monitoring of free plans, freemium limits, and paid-only APIs
- Lightweight compliance or operational visibility for automation teams

## 1.1 Input Reception and Runtime Configuration

The workflow starts manually and defines two runtime values:

- the base URL of the target n8n instance
- the recipient email address for the report

This block also relies on a pre-created HTTP Header Auth credential containing the n8n API key.

## 1.2 Credential Inventory Retrieval

The workflow calls the n8n REST API endpoint for credentials and fetches up to 250 credential records from the instance.

## 1.3 Service Name Extraction and Normalization

The returned credential types are converted into human-readable service names. Known internal or generic auth credential types are excluded, and duplicate services are removed.

## 1.4 AI Research Enrichment

A Claude model is connected to an LLM chain node. The chain receives the unique service list and is instructed to return only a JSON array containing pricing, free-tier, and rate-limit information for each service.

## 1.5 Report Formatting and Email Delivery

The AI response is parsed and transformed into a styled HTML table. Services are grouped visually by pricing tier and summarized in counts. The final report is sent to the configured email address via Gmail.

---

# 2. Block-by-Block Analysis

## 2.1 Input Reception and Runtime Configuration

### Overview

This block initializes the workflow and provides the base parameters needed downstream. It is intentionally simple so the workflow can be run manually or later replaced by a scheduled trigger.

### Nodes Involved

- Sticky Overview
- Sticky Setup
- Run Audit
- Configuration

### Node Details

#### Sticky Overview

- **Type and technical role:** `n8n-nodes-base.stickyNote`; documentation-only canvas annotation
- **Configuration choices:** Contains a functional summary of the workflow and mentions that it can be run manually or by schedule
- **Key expressions or variables used:** None
- **Input and output connections:** None
- **Version-specific requirements:** None
- **Edge cases or potential failure types:** No runtime impact
- **Sub-workflow reference:** None

#### Sticky Setup

- **Type and technical role:** `n8n-nodes-base.stickyNote`; setup guidance annotation
- **Configuration choices:** Describes the required HTTP Header Auth credential, instance URL placeholder replacement, notification email configuration, and Anthropic credential usage
- **Key expressions or variables used:** None
- **Input and output connections:** None
- **Version-specific requirements:** None
- **Edge cases or potential failure types:** No runtime impact
- **Sub-workflow reference:** None

#### Run Audit

- **Type and technical role:** `n8n-nodes-base.manualTrigger`; entry point
- **Configuration choices:** No parameters; execution begins when launched manually
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Output to `Configuration`
- **Version-specific requirements:** Type version 1
- **Edge cases or potential failure types:**
  - No automatic scheduling; must be started manually unless replaced or complemented by a trigger node
- **Sub-workflow reference:** None

#### Configuration

- **Type and technical role:** `n8n-nodes-base.set`; runtime parameter definition
- **Configuration choices:**
  - Creates `n8nInstanceUrl` as a string, currently set to `https://YOUR-INSTANCE.app.n8n.cloud`
  - Creates `notificationEmail` as a string, currently set to `your@email.com`
- **Key expressions or variables used:**
  - Exposes:
    - `$json.n8nInstanceUrl`
    - `$json.notificationEmail`
- **Input and output connections:**
  - Input from `Run Audit`
  - Output to `Get n8n Credentials`
- **Version-specific requirements:** Type version 3.4; uses assignments format from modern Set node UI
- **Edge cases or potential failure types:**
  - Placeholder URL not replaced
  - Invalid or incomplete instance URL
  - Invalid recipient email
  - Trailing slash in the instance URL is not fatal, but can produce double-slash endpoint formatting depending on value
- **Sub-workflow reference:** None

---

## 2.2 Credential Inventory Retrieval

### Overview

This block queries the n8n instance API to retrieve stored credentials. It depends on the base URL from the Configuration node and a valid API key stored in an HTTP Header Auth credential.

### Nodes Involved

- Get n8n Credentials

### Node Details

#### Get n8n Credentials

- **Type and technical role:** `n8n-nodes-base.httpRequest`; authenticated REST call to the local or remote n8n API
- **Configuration choices:**
  - URL is dynamically built from the configured instance URL plus `/api/v1/credentials?limit=250`
  - Authentication mode is `genericCredentialType`
  - Generic auth type is `httpHeaderAuth`
  - Uses credential named `n8n API Key`
- **Key expressions or variables used:**
  - `{{ $('Configuration').first().json.n8nInstanceUrl + '/api/v1/credentials?limit=250' }}`
- **Input and output connections:**
  - Input from `Configuration`
  - Output to `Extract Service Names`
- **Version-specific requirements:** Type version 4.2
- **Edge cases or potential failure types:**
  - Invalid or missing API key header
  - Endpoint inaccessible because of wrong instance URL
  - Permission issues if the API key lacks access
  - Network timeout or DNS failure
  - More than 250 credentials in the instance; the workflow does not paginate, so additional credentials would be omitted
  - Self-hosted instances with different base paths or reverse-proxy rules may require URL adjustment
- **Sub-workflow reference:** None

---

## 2.3 Service Name Extraction and Normalization

### Overview

This block interprets the raw credential payload returned by the n8n API. It maps credential types to service names, filters out generic auth mechanisms, deduplicates services, and prepares a comma-separated list for AI research.

### Nodes Involved

- Extract Service Names

### Node Details

#### Extract Service Names

- **Type and technical role:** `n8n-nodes-base.code`; JavaScript transformation node
- **Configuration choices:**
  - Reads the first input item
  - Supports either an object payload with `data` or a direct array
  - Uses a predefined `typeToService` mapping for many common n8n credential types
  - Excludes generic auth types by mapping them to `null`
  - Falls back to a formatted version of the credential type string if no explicit mapping exists
  - Deduplicates service names with a `Set`
- **Key expressions or variables used:**
  - Reads from `$input.first().json`
  - Returns:
    - `services`: array of `{ name, type }`
    - `serviceList`: comma-separated service names
    - `totalCreds`: total raw credential count
    - `uniqueServices`: count of deduplicated services
- **Input and output connections:**
  - Input from `Get n8n Credentials`
  - Output to `Research Pricing with Claude`
- **Version-specific requirements:** Type version 2
- **Edge cases or potential failure types:**
  - Unexpected API payload shape may produce an empty service list
  - Unknown credential types are prettified heuristically and may not match real service branding
  - Generic credentials such as `httpHeaderAuth` are intentionally discarded, which may hide integrations configured through generic HTTP nodes
  - A large number of services can create a long prompt for the LLM
  - If no credentials exist, the block still returns a valid item but with empty service data
- **Sub-workflow reference:** None

---

## 2.4 AI Research Enrichment

### Overview

This block sends the normalized service list to Claude and requests structured pricing metadata. The LLM chain is explicitly instructed to output only valid JSON and is backed by a dedicated Anthropic model node.

### Nodes Involved

- Research Pricing with Claude
- Claude Sonnet 4.5

### Node Details

#### Research Pricing with Claude

- **Type and technical role:** `@n8n/n8n-nodes-langchain.chainLlm`; prompt orchestration node for structured text generation
- **Configuration choices:**
  - Prompt type is `define`
  - Uses a main text prompt containing:
    - service count
    - service list
    - strict JSON output format
    - required fields for each service
  - Adds a system-style instruction via `messages.messageValues`:
    - “You are an API pricing expert. Return accurate, specific pricing and rate limit data. Return only the JSON array.”
  - Batch settings are present but not meaningfully customized
- **Key expressions or variables used:**
  - `{{ $json.uniqueServices }}`
  - `{{ $json.serviceList }}`
- **Input and output connections:**
  - Main input from `Extract Service Names`
  - AI language model input from `Claude Sonnet 4.5`
  - Main output to `Build HTML Report`
- **Version-specific requirements:** Type version 1.9; requires n8n LangChain-compatible AI nodes
- **Edge cases or potential failure types:**
  - LLM may return invalid JSON despite prompt constraints
  - Model may hallucinate pricing or provide stale information
  - Long service lists may exceed practical token limits or reduce output quality
  - Empty `serviceList` may produce unusable or trivial output
  - Anthropic credential issues or model access restrictions will stop execution
- **Sub-workflow reference:** None

#### Claude Sonnet 4.5

- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatAnthropic`; Anthropic chat model provider node
- **Configuration choices:**
  - Model selected by ID: `claude-sonnet-4-5-20250929`
  - Uses Anthropic credential named `Anthropic account`
  - No special model options configured
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Output via `ai_languageModel` to `Research Pricing with Claude`
- **Version-specific requirements:** Type version 1.3; requires a recent n8n version with LangChain/Anthropic support and model access for the specified Anthropic model ID
- **Edge cases or potential failure types:**
  - Invalid Anthropic API key
  - Selected model ID unavailable in the user account or region
  - Quota exhaustion or provider-side rate limiting
  - API/provider response errors
- **Sub-workflow reference:** None

---

## 2.5 Report Formatting and Email Delivery

### Overview

This block parses the Claude response, creates a styled HTML report with summary metrics and tier-based coloring, then sends the report to the configured recipient using Gmail.

### Nodes Involved

- Build HTML Report
- Email Audit Report

### Node Details

#### Build HTML Report

- **Type and technical role:** `n8n-nodes-base.code`; JavaScript formatter and fallback handler
- **Configuration choices:**
  - Reads `text` from the Claude chain output
  - Reads metadata from `Extract Service Names`
  - Removes possible Markdown code fences before attempting JSON parsing
  - If parsing fails, generates a fallback pseudo-record named `Parse Error`
  - Sorts records by `tier` order: `free`, `freemium`, `paid`
  - Generates HTML with:
    - title
    - generated date
    - credential/service counts
    - summary cards
    - formatted table
    - warning footer reminding users to verify official docs
- **Key expressions or variables used:**
  - `$input.first().json.text`
  - `$('Extract Service Names').first().json`
  - Output fields:
    - `html`
    - `totalCount`
    - `freeCount`
    - `freemiumCount`
    - `paidCount`
    - `date`
- **Input and output connections:**
  - Input from `Research Pricing with Claude`
  - Output to `Email Audit Report`
- **Version-specific requirements:** Type version 2
- **Edge cases or potential failure types:**
  - HTML escaping is not applied to model-generated content; malformed or unsafe text could affect rendering
  - If Claude returns partial records or missing fields, defaults are used but output quality may degrade
  - Parse fallback prevents total workflow failure, but email content may be less useful
  - Large reports may exceed comfortable email size limits
- **Sub-workflow reference:** None

#### Email Audit Report

- **Type and technical role:** `n8n-nodes-base.gmail`; outbound email sender
- **Configuration choices:**
  - Recipient is read from `Configuration.notificationEmail`
  - Subject includes service totals and tier counts with emoji
  - Email body is the HTML produced by the previous node
  - `appendAttribution` is disabled
  - Uses Gmail OAuth2 credential named `Gmail OAuth2 API`
- **Key expressions or variables used:**
  - `{{ $('Configuration').first().json.notificationEmail }}`
  - `{{ $json.html }}`
  - `{{ '🔌 Integration Audit: ' + $json.totalCount + ' services | 🟢 ' + $json.freeCount + ' Free | 🟡 ' + $json.freemiumCount + ' Freemium | 🔴 ' + $json.paidCount + ' Paid' }}`
- **Input and output connections:**
  - Input from `Build HTML Report`
  - No downstream node
- **Version-specific requirements:** Type version 2.1; requires Gmail OAuth2 credential configured in n8n
- **Edge cases or potential failure types:**
  - Gmail OAuth token expired or missing scopes
  - Gmail sending limits or anti-abuse restrictions
  - Invalid destination address
  - Some email clients may render the HTML differently
- **Sub-workflow reference:** None

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Overview | n8n-nodes-base.stickyNote | Canvas documentation summarizing workflow purpose and expected outcome |  |  | ## 🔌 n8n Integration Audit Report<br>**What this does:**<br>1. Reads all credentials connected to your n8n instance<br>2. Maps credential types to real service names<br>3. Sends the list to Claude AI — researches pricing, free tiers, and rate limits for each<br>4. Emails a colour-coded HTML report: 🟢 Free · 🟡 Freemium · 🔴 Paid<br><br>**Run manually** or connect a Schedule trigger for weekly reports. |
| Sticky Setup | n8n-nodes-base.stickyNote | Canvas documentation for setup prerequisites |  |  | ## ⚙️ Setup — 3 steps<br>**1. n8n API Key credential**<br>Create an **HTTP Header Auth** credential:<br>- Name: `n8n API Key`<br>- Header Name: `X-N8N-API-KEY`<br>- Header Value: your n8n API key<br>(Settings → API → Create API Key)<br><br>**2. Instance URL**<br>In the **Configuration** node, replace the placeholder with your n8n instance URL.<br>Example: `https://myname.app.n8n.cloud`<br><br>**3. Notification email**<br>In the **Configuration** node, set your email address.<br><br>**Anthropic credential** is used for Claude AI pricing research. |
| Run Audit | n8n-nodes-base.manualTrigger | Manual workflow entry point |  | Configuration | ## 🔌 n8n Integration Audit Report<br>**What this does:**<br>1. Reads all credentials connected to your n8n instance<br>2. Maps credential types to real service names<br>3. Sends the list to Claude AI — researches pricing, free tiers, and rate limits for each<br>4. Emails a colour-coded HTML report: 🟢 Free · 🟡 Freemium · 🔴 Paid<br><br>**Run manually** or connect a Schedule trigger for weekly reports.<br>## ⚙️ Setup — 3 steps<br>**1. n8n API Key credential**<br>Create an **HTTP Header Auth** credential:<br>- Name: `n8n API Key`<br>- Header Name: `X-N8N-API-KEY`<br>- Header Value: your n8n API key<br>(Settings → API → Create API Key)<br><br>**2. Instance URL**<br>In the **Configuration** node, replace the placeholder with your n8n instance URL.<br>Example: `https://myname.app.n8n.cloud`<br><br>**3. Notification email**<br>In the **Configuration** node, set your email address.<br><br>**Anthropic credential** is used for Claude AI pricing research. |
| Configuration | n8n-nodes-base.set | Defines instance URL and notification email | Run Audit | Get n8n Credentials | ## ⚙️ Setup — 3 steps<br>**1. n8n API Key credential**<br>Create an **HTTP Header Auth** credential:<br>- Name: `n8n API Key`<br>- Header Name: `X-N8N-API-KEY`<br>- Header Value: your n8n API key<br>(Settings → API → Create API Key)<br><br>**2. Instance URL**<br>In the **Configuration** node, replace the placeholder with your n8n instance URL.<br>Example: `https://myname.app.n8n.cloud`<br><br>**3. Notification email**<br>In the **Configuration** node, set your email address.<br><br>**Anthropic credential** is used for Claude AI pricing research. |
| Get n8n Credentials | n8n-nodes-base.httpRequest | Fetches credential inventory from the n8n API | Configuration | Extract Service Names | ## ⚙️ Setup — 3 steps<br>**1. n8n API Key credential**<br>Create an **HTTP Header Auth** credential:<br>- Name: `n8n API Key`<br>- Header Name: `X-N8N-API-KEY`<br>- Header Value: your n8n API key<br>(Settings → API → Create API Key)<br><br>**2. Instance URL**<br>In the **Configuration** node, replace the placeholder with your n8n instance URL.<br>Example: `https://myname.app.n8n.cloud`<br><br>**3. Notification email**<br>In the **Configuration** node, set your email address.<br><br>**Anthropic credential** is used for Claude AI pricing research. |
| Extract Service Names | n8n-nodes-base.code | Maps credential types to deduplicated service names | Get n8n Credentials | Research Pricing with Claude |  |
| Research Pricing with Claude | @n8n/n8n-nodes-langchain.chainLlm | Uses Claude to enrich services with pricing and limits data | Extract Service Names, Claude Sonnet 4.5 | Build HTML Report |  |
| Claude Sonnet 4.5 | @n8n/n8n-nodes-langchain.lmChatAnthropic | Anthropic language model provider for the LLM chain |  | Research Pricing with Claude |  |
| Build HTML Report | n8n-nodes-base.code | Parses AI output and builds the HTML email report | Research Pricing with Claude | Email Audit Report |  |
| Email Audit Report | n8n-nodes-base.gmail | Sends the final HTML report via Gmail | Build HTML Report |  |  |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it something like: `n8n Integration Audit — Pricing, Tiers & Limits Report`.

2. **Add a Manual Trigger node**
   - Node type: **Manual Trigger**
   - Name: `Run Audit`
   - Leave default configuration.

3. **Add a Set node for configuration**
   - Node type: **Set**
   - Name: `Configuration`
   - Add two string fields:
     1. `n8nInstanceUrl` = your n8n instance URL, for example `https://myname.app.n8n.cloud`
     2. `notificationEmail` = recipient address for the report
   - Connect `Run Audit` → `Configuration`.

4. **Create the n8n API credential**
   - Go to **Credentials**.
   - Create a credential of type **HTTP Header Auth**.
   - Recommended name: `n8n API Key`
   - Header name: `X-N8N-API-KEY`
   - Header value: your n8n API key from **Settings → API**
   - Save it.

5. **Add an HTTP Request node to fetch credentials**
   - Node type: **HTTP Request**
   - Name: `Get n8n Credentials`
   - Method: `GET`
   - URL:
     ```text
     {{ $('Configuration').first().json.n8nInstanceUrl + '/api/v1/credentials?limit=250' }}
     ```
   - Authentication: **Generic Credential Type**
   - Generic Auth Type: **HTTP Header Auth**
   - Select the credential `n8n API Key`
   - Keep response as JSON
   - Connect `Configuration` → `Get n8n Credentials`.

6. **Add a Code node to normalize service names**
   - Node type: **Code**
   - Name: `Extract Service Names`
   - Language: JavaScript
   - Paste this logic conceptually:
     - read the HTTP response
     - use `body.data` if present, otherwise use the response directly if it is an array
     - map known credential types to human-friendly service names
     - skip generic auth types such as `httpHeaderAuth`, `httpBasicAuth`, `oAuth2Api`, etc.
     - deduplicate service names
     - output:
       - `services`
       - `serviceList`
       - `totalCreds`
       - `uniqueServices`
   - Use the following code:

```javascript
const body = $input.first().json;
const creds = body.data || (Array.isArray(body) ? body : []);

const typeToService = {
  openAiApi: 'OpenAI', anthropicApi: 'Anthropic (Claude AI)',
  googleDriveOAuth2Api: 'Google Drive', gmailOAuth2: 'Gmail',
  googleSheetsOAuth2Api: 'Google Sheets', googleCalendarOAuth2Api: 'Google Calendar',
  googleDocsOAuth2Api: 'Google Docs', youtubeOAuth2Api: 'YouTube Data API',
  slackApi: 'Slack', slackOAuth2Api: 'Slack', telegramApi: 'Telegram Bot API',
  notionApi: 'Notion', airtableTokenApi: 'Airtable', airtableApi: 'Airtable',
  githubApi: 'GitHub', gitlabApi: 'GitLab', hubspotApi: 'HubSpot',
  salesforceOAuth2Api: 'Salesforce', stripeApi: 'Stripe', twilioApi: 'Twilio',
  sendGridApi: 'SendGrid', mailchimpApi: 'Mailchimp', shopifyApi: 'Shopify',
  dropboxApi: 'Dropbox', oneDriveApi: 'Microsoft OneDrive',
  microsoftTeamsOAuth2Api: 'Microsoft Teams', outlookOAuth2Api: 'Microsoft Outlook',
  linkedInOAuth2Api: 'LinkedIn', twitterOAuth2Api: 'Twitter / X',
  facebookGraphApi: 'Facebook Graph API', wordpressApi: 'WordPress REST API',
  asanaApi: 'Asana', trelloApi: 'Trello', jiraApi: 'Jira (Atlassian)',
  linearApi: 'Linear', clickupApi: 'ClickUp', mondayApi: 'Monday.com',
  pipedriveApi: 'Pipedrive', zendeskApi: 'Zendesk', discordApi: 'Discord',
  discordBotApi: 'Discord Bot', postgresDb: 'PostgreSQL', mysqlDb: 'MySQL',
  mongoDb: 'MongoDB', supabaseApi: 'Supabase', pineconeApi: 'Pinecone',
  httpHeaderAuth: null, httpBasicAuth: null, httpQueryAuth: null,
  httpDigestAuth: null, oAuth2Api: null, oAuth1Api: null,
  sshPrivateKey: null, noAuth: null,
};

const seen = new Set();
const services = [];
for (const cred of creds) {
  const mapped = typeToService[cred.type];
  if (mapped === null) continue;
  const name = mapped || cred.type.replace(/([A-Z])/g, ' $1').replace('Api', ' API').trim();
  if (!seen.has(name)) { seen.add(name); services.push({ name, type: cred.type }); }
}

return [{ json: { services, serviceList: services.map(s => s.name).join(', '), totalCreds: creds.length, uniqueServices: services.length } }];
```

   - Connect `Get n8n Credentials` → `Extract Service Names`.

7. **Create an Anthropic credential**
   - Go to **Credentials**
   - Create **Anthropic API** credential
   - Save it with a name such as `Anthropic account`
   - Ensure the API key has access to the intended Claude model.

8. **Add an Anthropic chat model node**
   - Node type: **Anthropic Chat Model** / `lmChatAnthropic`
   - Name: `Claude Sonnet 4.5`
   - Select the Anthropic credential
   - Choose the model ID:
     - `claude-sonnet-4-5-20250929`
   - Leave optional settings default unless you need temperature or token changes.

9. **Add an LLM Chain node**
   - Node type: **Basic LLM Chain** / `chainLlm`
   - Name: `Research Pricing with Claude`
   - Prompt type: **Define**
   - Main prompt text:

```text
Analyze these {{ $json.uniqueServices }} services and return their API pricing, free tiers, and rate limits.

Services: {{ $json.serviceList }}

Return ONLY a valid JSON array, no markdown, no explanation:
[
  {
    "service": "service name",
    "category": "AI | Storage | Communication | CRM | Productivity | Social | Database | Payment | Other",
    "tier": "free | freemium | paid",
    "hasFree": true or false,
    "freeLimits": "specific quota e.g. 10K chars/month",
    "paidFrom": "lowest price e.g. $10/month",
    "rateLimit": "e.g. 60 RPM or No published limit",
    "notes": "1-2 sentences on key restrictions"
  }
]
```

   - Add an instruction message:
     - `You are an API pricing expert. Return accurate, specific pricing and rate limit data. Return only the JSON array.`
   - Connect:
     - `Extract Service Names` → `Research Pricing with Claude`
     - `Claude Sonnet 4.5` → `Research Pricing with Claude` using the **AI language model** connection.

10. **Add a Code node to build the HTML report**
    - Node type: **Code**
    - Name: `Build HTML Report`
    - Language: JavaScript
    - Use code that:
      - reads `text` from the chain output
      - strips markdown fences if present
      - parses JSON
      - on failure, creates a fallback error record
      - sorts by pricing tier
      - builds an HTML table and summary cards
      - returns `html`, `totalCount`, `freeCount`, `freemiumCount`, `paidCount`, `date`
    - Use this code:

```javascript
const rawText = $input.first().json.text || '';
const meta = $('Extract Service Names').first().json;

let integrations = [];
try {
  const cleaned = rawText.replace(/```json\n?/g,'').replace(/```\n?/g,'').trim();
  integrations = JSON.parse(cleaned);
} catch(e) {
  integrations = [{ service:'Parse Error', category:'Error', tier:'freemium', hasFree:false, freeLimits:'Could not parse response', paidFrom:'N/A', rateLimit:'N/A', notes: rawText.substring(0,300) }];
}

const tierOrder = { free:0, freemium:1, paid:2 };
integrations.sort((a,b) => {
  const ta = tierOrder[a.tier]??1, tb = tierOrder[b.tier]??1;
  return ta!==tb ? ta-tb : a.service.localeCompare(b.service);
});

const tierColor={free:'#16a34a',freemium:'#b45309',paid:'#dc2626'};
const tierBg={free:'#dcfce7',freemium:'#fef3c7',paid:'#fee2e2'};
const tierIcon={free:'🟢',freemium:'🟡',paid:'🔴'};

const rows = integrations.map((item,i) => {
  const t = item.tier||'freemium';
  const bg = i%2===0?'#ffffff':'#f9fafb';
  return `<tr style="background:${bg};border-bottom:1px solid #e5e7eb;">
    <td style="padding:10px 14px;font-weight:600;">${item.service}</td>
    <td style="padding:10px 14px;color:#6b7280;font-size:13px;">${item.category||''}</td>
    <td style="padding:10px 14px;"><span style="background:${tierBg[t]};color:${tierColor[t]};padding:3px 10px;border-radius:12px;font-size:12px;font-weight:700;">${tierIcon[t]} ${t.charAt(0).toUpperCase()+t.slice(1)}</span></td>
    <td style="padding:10px 14px;font-size:13px;">${item.freeLimits||(item.hasFree?'Yes':'❌ None')}</td>
    <td style="padding:10px 14px;font-size:13px;">${item.paidFrom||'N/A'}</td>
    <td style="padding:10px 14px;font-size:13px;">${item.rateLimit||'N/A'}</td>
    <td style="padding:10px 14px;font-size:12px;color:#6b7280;">${item.notes||''}</td>
  </tr>`;
}).join('');

const date = new Date().toLocaleDateString('en-US',{year:'numeric',month:'long',day:'numeric'});
const freeCount=integrations.filter(i=>i.tier==='free').length;
const freemiumCount=integrations.filter(i=>i.tier==='freemium').length;
const paidCount=integrations.filter(i=>i.tier==='paid').length;

const html=`<div style="font-family:Arial,sans-serif;max-width:1100px;margin:0 auto;padding:20px;">
  <h1 style="color:#0F1117;border-bottom:4px solid #EA4B71;padding-bottom:14px;">🔌 n8n Integration Audit Report</h1>
  <p style="color:#6b7280;">Generated: ${date} · ${meta.totalCreds} credentials · ${integrations.length} services analysed</p>
  <div style="display:flex;gap:12px;margin:16px 0 24px;">
    <div style="background:#dcfce7;color:#16a34a;padding:10px 18px;border-radius:8px;font-weight:700;">🟢 Free: ${freeCount}</div>
    <div style="background:#fef3c7;color:#b45309;padding:10px 18px;border-radius:8px;font-weight:700;">🟡 Freemium: ${freemiumCount}</div>
    <div style="background:#fee2e2;color:#dc2626;padding:10px 18px;border-radius:8px;font-weight:700;">🔴 Paid only: ${paidCount}</div>
  </div>
  <table style="border-collapse:collapse;width:100%;font-size:14px;border:1px solid #e5e7eb;">
    <thead><tr style="background:#0F1117;color:#fff;">
      <th style="padding:12px 14px;text-align:left;">Service</th><th style="padding:12px 14px;text-align:left;">Category</th>
      <th style="padding:12px 14px;text-align:left;">Tier</th><th style="padding:12px 14px;text-align:left;">Free Limits</th>
      <th style="padding:12px 14px;text-align:left;">Paid From</th><th style="padding:12px 14px;text-align:left;">Rate Limits</th>
      <th style="padding:12px 14px;text-align:left;">Notes</th>
    </tr></thead>
    <tbody>${rows}</tbody>
  </table>
  <p style="color:#9ca3af;font-size:12px;margin-top:16px;">⚠️ Pricing researched by Claude AI — verify with official docs.</p>
</div>`;

return [{ json:{ html, totalCount:integrations.length, freeCount, freemiumCount, paidCount, date } }];
```

   - Connect `Research Pricing with Claude` → `Build HTML Report`.

11. **Create a Gmail OAuth2 credential**
   - Go to **Credentials**
   - Create **Gmail OAuth2**
   - Authenticate the Gmail account that will send the report
   - Ensure send-email capability is authorized.

12. **Add a Gmail node**
   - Node type: **Gmail**
   - Name: `Email Audit Report`
   - Operation: send email
   - To:
     ```text
     {{ $('Configuration').first().json.notificationEmail }}
     ```
   - Subject:
     ```text
     {{ '🔌 Integration Audit: ' + $json.totalCount + ' services | 🟢 ' + $json.freeCount + ' Free | 🟡 ' + $json.freemiumCount + ' Freemium | 🔴 ' + $json.paidCount + ' Paid' }}
     ```
   - Message/body:
     ```text
     {{ $json.html }}
     ```
   - In options, disable attribution if available:
     - `appendAttribution: false`
   - Select the Gmail OAuth2 credential
   - Connect `Build HTML Report` → `Email Audit Report`.

13. **Optionally add sticky notes**
   - Add one sticky note explaining the workflow purpose
   - Add another sticky note describing setup:
     - n8n API Key credential
     - instance URL update
     - notification email update
     - Anthropic credential requirement

14. **Save and test**
   - Run the workflow manually
   - Confirm:
     - the HTTP Request returns credential data
     - the service extraction list is populated
     - Claude returns JSON text
     - the HTML is generated
     - the Gmail node sends successfully

15. **Optional production improvement**
   - Replace or supplement `Run Audit` with a **Schedule Trigger** for weekly delivery
   - Consider pagination for `/api/v1/credentials` if your instance exceeds 250 credentials
   - Consider storing report history in Sheets, Notion, or a database

### Required Credentials Summary

- **HTTP Header Auth**
  - Name: `n8n API Key`
  - Header name: `X-N8N-API-KEY`
  - Header value: n8n API key

- **Anthropic API**
  - Used by `Claude Sonnet 4.5`

- **Gmail OAuth2**
  - Used by `Email Audit Report`

### Input/Output Expectations

- **HTTP Request output:** JSON object with `data` array of credential objects, or array-like response
- **Extract Service Names output:** single item with service metadata
- **LLM Chain output:** text field containing a valid JSON array as plain text
- **Build HTML Report output:** single item containing HTML and count summaries
- **Final output:** email sent via Gmail

### There are no sub-workflows in this workflow

- No Execute Workflow node is present
- No separate child workflow invocation is required

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| The workflow is inactive by default (`active: false`) and must be manually run or explicitly activated with a trigger strategy. | Workflow runtime behavior |
| The AI-generated pricing data is explicitly flagged in the email as requiring verification against official vendor documentation. | Reporting caveat |
| The workflow currently fetches only the first 250 credentials from the n8n API. If the instance contains more, pagination should be added. | n8n API limitation |
| Generic auth credential types are intentionally excluded from service reporting, which can undercount integrations built through generic HTTP authentication rather than service-specific credential types. | Service extraction logic |
| The selected Claude model ID is `claude-sonnet-4-5-20250929`; availability depends on the Anthropic account and n8n node compatibility. | Anthropic model configuration |