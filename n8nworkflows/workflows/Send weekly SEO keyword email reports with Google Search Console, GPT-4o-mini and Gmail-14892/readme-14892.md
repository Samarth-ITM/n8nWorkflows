Send weekly SEO keyword email reports with Google Search Console, GPT-4o-mini and Gmail

https://n8nworkflows.xyz/workflows/send-weekly-seo-keyword-email-reports-with-google-search-console--gpt-4o-mini-and-gmail-14892


# Send weekly SEO keyword email reports with Google Search Console, GPT-4o-mini and Gmail

Let me now create the comprehensive document based on my analysis. 1. Workflow Overview

This workflow automates the generation and delivery of a **weekly SEO keyword performance email report** for agency clients. Every Monday at 8:00 AM, it:

1. Triggers on a cron schedule.
2. Loads static configuration values (site URL, client name, recipient email, agency name).
3. Calls the **Google Search Console API** via an HTTP request to retrieve the top 10 search queries from the past 7 days, ordered by clicks.
4. Extracts and formats the raw API response into a human-readable keyword summary string.
5. Passes the formatted data to a **GPT-4o-mini AI Agent** that writes a professional 150–200 word email body.
6. Assembles the final email (subject, recipient, body) and sends it through **Gmail**.

**Logical Blocks:**

| Block | Nodes | Purpose |
|---|---|---|
| **1.1 Schedule and Config** | Schedule Trigger, Set — Config Values | Weekly trigger and static configuration injection |
| **1.2 GSC Data Collection** | HTTP — Fetch GSC Top Keywords, Set — Extract Fields, Code — Format Data for GPT | Query Google Search Console, extract clean fields, format for AI |
| **1.3 AI Report Writing** | AI Agent — Write SEO Report Email, OpenAI — GPT-4o-mini Model | Generate the email body via GPT-4o-mini |
| **1.4 Email Delivery** | Set — Prepare Final Email, Gmail — Send Weekly Report | Assemble final email payload and send via Gmail |

---

### 2. Block-by-Block Analysis

---

#### Block 1.1 — Schedule and Config

**Overview:** This block is the entry point. A cron-based schedule triggers the workflow every Monday at 08:00. The subsequent Set node injects four static configuration values (site URL, client name, recipient email, agency name) that are referenced by every downstream node.

**Nodes Involved:**
- 1. Schedule — Every Monday 8AM
- 2. Set — Config Values

---

**Node: 1. Schedule — Every Monday 8AM**

| Property | Detail |
|---|---|
| **Type** | `n8n-nodes-base.scheduleTrigger` (v1.2) |
| **Technical Role** | Workflow trigger; produces one empty item per execution |
| **Configuration** | Cron expression: `0 8 * * 1` (every Monday at 08:00 in the server timezone) |
| **Key Expressions** | None |
| **Input Connections** | None (entry point) |
| **Output Connections** | → `2. Set — Config Values` (main, index 0) |
| **Edge Cases** | If the n8n server timezone differs from the intended one, the email may be sent at a different local time. Daylight Saving Time shifts may cause the trigger to fire an hour off if the server TZ is not UTC-based. No retries on failure — the schedule simply fires again next week. |

---

**Node: 2. Set — Config Values**

| Property | Detail |
|---|---|
| **Type** | `n8n-nodes-base.set` (v3.4) |
| **Technical Role** | Static variable injection; defines all customizable parameters in one place |
| **Configuration** | Four string assignments: |
| | • `siteUrl` = `https://www.YOUR-WEBSITE.com/` — must exactly match the GSC property URL |
| | • `clientName` = `YOUR CLIENT NAME` — client business name, used in the email body and subject |
| | • `recipientEmail` = `user@example.com` — destination email address |
| | • `agencyName` = `YOUR AGENCY NAME` — appears in the AI prompt and email footer |
| **Key Expressions** | All values are static literals (no expressions). They are referenced downstream via `$('2. Set — Config Values').item.json.<fieldName>`. |
| **Input Connections** | ← `1. Schedule — Every Monday 8AM` (main, index 0) |
| **Output Connections** | → `3. HTTP — Fetch GSC Top Keywords` (main, index 0) |
| **Edge Cases** | If `siteUrl` does not exactly match a verified GSC property (including trailing slash), the API call in node 3 will return a 403 or 404. An invalid email address in `recipientEmail` will cause a Gmail send failure in node 9. Leading/trailing whitespace in any value will propagate silently. |

---

#### Block 1.2 — GSC Data Collection

**Overview:** This block queries the Google Search Console API for the top 10 search queries over the last 7 days, extracts the needed fields alongside the config values, and transforms the raw API rows into a formatted text string suitable for the AI model.

**Nodes Involved:**
- 3. HTTP — Fetch GSC Top Keywords
- 4. Set — Extract Fields
- 5. Code — Format Data for GPT

---

**Node: 3. HTTP — Fetch GSC Top Keywords**

| Property | Detail |
|---|---|
| **Type** | `n8n-nodes-base.httpRequest` (v4.2) |
| **Technical Role** | Calls the GSC Search Analytics API endpoint to retrieve keyword performance data |
| **Configuration** | |
| | • **URL** (expression): `https://www.googleapis.com/webmasters/v3/sites/{{ encodeURIComponent($json.siteUrl) }}/searchAnalytics/query` — the `siteUrl` from node 2 is URL-encoded automatically |
| | • **Method**: POST |
| | • **Body** (JSON, expression): |
| |   - `startDate`: `$now.minus({days: 7}).toFormat('yyyy-MM-dd')` — 7 days ago |
| |   - `endDate`: `$now.minus({days: 1}).toFormat('yyyy-MM-dd')` — yesterday |
| |   - `dimensions`: `["query"]` — group by search query |
| |   - `rowLimit`: `10` |
| |   - `orderby`: `[{"fieldName":"clicks","sortOrder":"DESCENDING"}]` — top 10 by clicks |
| | • **Headers**: `Content-Type: application/json` |
| | • **Authentication**: OAuth2 (Google Search Console credential required) |
| **Key Expressions** | `encodeURIComponent($json.siteUrl)`, `$now.minus({days: 7}).toFormat('yyyy-MM-dd')`, `$now.minus({days: 1}).toFormat('yyyy-MM-dd')` |
| **Input Connections** | ← `2. Set — Config Values` (main, index 0) |
| **Output Connections** | → `4. Set — Extract Fields` (main, index 0) |
| **Edge Cases** | **403 Forbidden** if the OAuth2 credential lacks the `https://www.googleapis.com/auth/webmasters.readonly` scope or if the site is not verified in GSC. **404 Not Found** if `siteUrl` encoding is wrong (e.g., missing `encodeURIComponent`). **Empty `rows` array** if no data exists for the date range (the workflow handles this gracefully in node 5). Timeout on slow Google API responses (default HTTP timeout applies). If the GSC property has very low traffic, fewer than 10 rows may be returned. |

---

**Node: 4. Set — Extract Fields**

| Property | Detail |
|---|---|
| **Type** | `n8n-nodes-base.set` (v3.4) |
| **Technical Role** | Pulls configuration values from node 2 and GSC response data into clean named fields on the current item |
| **Configuration** | Seven assignments: |
| | • `siteUrl` ← `$('2. Set — Config Values').item.json.siteUrl` |
| | • `clientName` ← `$('2. Set — Config Values').item.json.clientName` |
| | • `recipientEmail` ← `$('2. Set — Config Values').item.json.recipientEmail` |
| | • `agencyName` ← `$('2. Set — Config Values').item.json.agencyName` |
| | • `weekStart` ← `$now.minus({days: 7}).toFormat('dd MMM yyyy')` — human-readable start date |
| | • `weekEnd` ← `$now.minus({days: 1}).toFormat('dd MMM yyyy')` — human-readable end date |
| | • `rawRows` ← `JSON.stringify($json.rows || [])` — serializes the GSC rows array into a string for the Code node |
| **Key Expressions** | References to node 2 via `$('2. Set — Config Values').item.json.*`; date formatting with Luxon's `toFormat`; `JSON.stringify` with fallback to `[]` |
| **Input Connections** | ← `3. HTTP — Fetch GSC Top Keywords` (main, index 0) |
| **Output Connections** | → `5. Code — Format Data for GPT` (main, index 0) |
| **Edge Cases** | If the GSC API returns no `rows` property (e.g., an error response), `$json.rows` is `undefined` and the fallback `[]` produces an empty array. The `weekStart`/`weekEnd` dates are computed at execution time — they will differ by a day if the workflow runs at midnight boundaries. |

---

**Node: 5. Code — Format Data for GPT**

| Property | Detail |
|---|---|
| **Type** | `n8n-nodes-base.code` (v2) |
| **Technical Role** | Transforms the serialized GSC JSON rows into a numbered, human-readable text block; also forwards all config fields |
| **Configuration** | Single JavaScript code block (key logic below): |
| | 1. Parses `rawRows` from the input item (with try/catch defaulting to `[]`) |
| | 2. If `rows.length === 0`, sets `keywordText` to a warning message |
| | 3. Otherwise iterates over each row, extracting: `query` (from `row.keys[0]`), `clicks`, `impressions`, `ctr` (converted to percentage with 1 decimal), `position` (1 decimal) |
| | 4. Builds a numbered string: `"1. \"keyword\" | Clicks: X | Impressions: Y | CTR: Z% | Position: P"` |
| | 5. Returns a single item with fields: `clientName`, `recipientEmail`, `agencyName`, `weekStart`, `weekEnd`, `keywordText`, `totalKeywords` |
| **Key Expressions** | All logic is in JavaScript; no n8n expressions are used. The `ctr` field from GSC is a decimal (e.g., `0.045` = 4.5%), so it's multiplied by 100. |
| **Input Connections** | ← `4. Set — Extract Fields` (main, index 0) |
| **Output Connections** | → `6. AI Agent — Write SEO Report Email` (main, index 0) |
| **Edge Cases** | If `row.keys` is missing or empty, the query defaults to `'unknown'`. If `row.ctr` or `row.position` is `undefined`, they default to `0`. A `JSON.parse` failure on `rawRows` is caught silently and defaults to an empty array. If GSC returns fewer than 10 rows, only the available rows are listed. |

---

#### Block 1.3 — AI Report Writing

**Overview:** This block uses an n8n AI Agent powered by GPT-4o-mini to generate a professional, plain-text email body. The agent receives the formatted keyword data and strict writing instructions. The OpenAI model node is connected as a language model sub-node to the agent.

**Nodes Involved:**
- 6. AI Agent — Write SEO Report Email
- 7. OpenAI — GPT-4o-mini Model

---

**Node: 6. AI Agent — Write SEO Report Email**

| Property | Detail |
|---|---|
| **Type** | `@n8n/n8n-nodes-langchain.agent` (v1.7) |
| **Technical Role** | AI agent that generates the email body text based on the keyword data and prompt |
| **Configuration** | |
| | • **Prompt Type**: `define` (prompt defined inline in the node, not via a separate prompt node) |
| | • **Text** (expression): Full prompt composed of: |
| |   - System-like preamble: "You are a professional SEO report writer working for {{ $json.agencyName }}." |
| |   - Context: CLIENT NAME, REPORT PERIOD (weekStart–weekEnd), TOP KEYWORDS DATA (keywordText) |
| |   - Writing instructions: plain text only (no markdown, no asterisks, no bullet symbols, no hashtags), 150–200 words, four specific paragraphs (greeting, top 3 keywords highlight, positive observation, actionable tip), professional sign-off ending with "Best regards," |
| | • **Options**: None (default) |
| **Key Expressions** | `{{ $json.agencyName }}`, `{{ $json.clientName }}`, `{{ $json.weekStart }}`, `{{ $json.weekEnd }}`, `{{ $json.keywordText }}` |
| **Input Connections** | ← `5. Code — Format Data for GPT` (main, index 0); ← `7. OpenAI — GPT-4o-mini Model` (ai_languageModel, index 0) |
| **Output Connections** | → `8. Set — Prepare Final Email` (main, index 0) |
| **Edge Cases** | If OpenAI returns an error (quota exceeded, invalid API key, rate limit), the agent node will throw and the workflow will fail. The `output` field in the agent's JSON result may be empty or missing — node 8 handles this with a fallback string. If GPT-4o-mini generates markdown despite instructions, it will appear verbatim in the email. The prompt constrains length but the model may occasionally exceed 200 words. |

---

**Node: 7. OpenAI — GPT-4o-mini Model**

| Property | Detail |
|---|---|
| **Type** | `@n8n/n8n-nodes-langchain.lmChatOpenAi` (v1.2) |
| **Technical Role** | Language model sub-node; provides GPT-4o-mini as the AI backend for the agent |
| **Configuration** | |
| | • **Model**: `gpt-4o-mini` (selected from list mode) |
| | • **Max Tokens**: 500 |
| | • **Temperature**: 0.6 |
| | • **Credential**: OpenAI API key credential (referenced as "testing" — must be configured) |
| **Key Expressions** | None |
| **Input Connections** | None (sub-node, connected via ai_languageModel) |
| **Output Connections** | → `6. AI Agent — Write SEO Report Email` (ai_languageModel, index 0) |
| **Edge Cases** | Invalid or expired OpenAI API key causes authentication failure. Rate limiting on the OpenAI side may require retries. If the OpenAI account has insufficient quota, the call will fail. The `maxTokens` of 500 is well above the 150–200 word target, ensuring the model does not truncate. Temperature of 0.6 balances creativity with consistency. |

---

#### Block 1.4 — Email Delivery

**Overview:** This block assembles the final email payload (recipient, subject, body) from the AI agent output and the previously stored config values, then sends the email via Gmail with an auto-generated attribution footer.

**Nodes Involved:**
- 8. Set — Prepare Final Email
- 9. Gmail — Send Weekly Report

---

**Node: 8. Set — Prepare Final Email**

| Property | Detail |
|---|---|
| **Type** | `n8n-nodes-base.set` (v3.4) |
| **Technical Role** | Maps the AI-generated text and config values into fields expected by the Gmail node |
| **Configuration** | Four assignments: |
| | • `toEmail` ← `$('5. Code — Format Data for GPT').item.json.recipientEmail` — pulls from the Code node's output |
| | • `emailSubject` ← `"Weekly SEO Report — {{ weekStart }} to {{ weekEnd }} | {{ clientName }}"` — dynamic subject line referencing the Code node |
| | • `emailBody` ← `$json.output || 'Email content could not be generated. Please check OpenAI credentials.'` — AI agent's text output with fallback |
| | • `agencyName` ← `$('5. Code — Format Data for GPT').item.json.agencyName` — for the footer |
| **Key Expressions** | References to node 5 via `$('5. Code — Format Data for GPT').item.json.*`; `$json.output` with fallback |
| **Input Connections** | ← `6. AI Agent — Write SEO Report Email` (main, index 0) |
| **Output Connections** | → `9. Gmail — Send Weekly Report` (main, index 0) |
| **Edge Cases** | If the AI agent's output structure changes (e.g., the field is named `text` instead of `output`), the fallback message will be sent as the email body. If `recipientEmail` from node 5 is blank or malformed, the Gmail node will fail. The subject line could be very long if `clientName` or date strings are lengthy, but Gmail supports subjects up to ~990 characters. |

---

**Node: 9. Gmail — Send Weekly Report**

| Property | Detail |
|---|---|
| **Type** | `n8n-nodes-base.gmail` (v2.1) |
| **Technical Role** | Sends the final email to the client via Gmail |
| **Configuration** | |
| | • **To** (expression): `{{ $json.toEmail }}` |
| | • **Subject** (expression): `{{ $json.emailSubject }}` |
| | • **Message** (expression): The email body followed by a footer: `\n\n--\nThis report was auto-generated by {{ $json.agencyName }}.\nData source: Google Search Console | Powered by n8n + GPT-4o-mini` |
| | • **Options**: `appendAttribution: false` — disables n8n's default attribution |
| | • **Credential**: Gmail OAuth2 credential (referenced as "support@entellustech.com" — must be configured) |
| **Key Expressions** | `{{ $json.toEmail }}`, `{{ $json.emailSubject }}`, `{{ $json.emailBody }}`, `{{ $json.agencyName }}` |
| **Input Connections** | ← `8. Set — Prepare Final Email` (main, index 0) |
| **Output Connections** | None (terminal node) |
| **Edge Cases** | Gmail API daily send limit (typically 500 for regular Gmail, 2000 for Workspace) — unlikely to be hit with a single weekly email. OAuth2 token expiration — the credential must have refresh token capability. If the recipient address bounces, Gmail returns an error. If `appendAttribution` were left as default (true), an additional n8n branding line would appear in the email. |

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| 1. Schedule — Every Monday 8AM | scheduleTrigger (v1.2) | Weekly cron trigger at Monday 08:00 | — | 2. Set — Config Values | Schedule and Config: Workflow triggers every Monday at 8AM. Config values — site URL, client name, recipient email, and agency name — are set here once and used throughout. |
| 2. Set — Config Values | set (v3.4) | Inject static config values (siteUrl, clientName, recipientEmail, agencyName) | 1. Schedule — Every Monday 8AM | 3. HTTP — Fetch GSC Top Keywords | Schedule and Config: Workflow triggers every Monday at 8AM. Config values — site URL, client name, recipient email, and agency name — are set here once and used throughout. ⚠️ Edit This Node Before Activating: This is the only node you need to change. Replace all four values before running: siteUrl must match your exact GSC property URL, clientName is the client business name, recipientEmail is where the report goes, and agencyName appears in the email footer. |
| 3. HTTP — Fetch GSC Top Keywords | httpRequest (v4.2) | Call Google Search Console Search Analytics API for top 10 keywords | 2. Set — Config Values | 4. Set — Extract Fields | GSC Data Collection: Fetches the top 10 keywords from Google Search Console for the past 7 days. Extracts and formats the raw API response into clean readable text for the AI. |
| 4. Set — Extract Fields | set (v3.4) | Extract config and GSC row data into clean named fields | 3. HTTP — Fetch GSC Top Keywords | 5. Code — Format Data for GPT | GSC Data Collection: Fetches the top 10 keywords from Google Search Console for the past 7 days. Extracts and formats the raw API response into clean readable text for the AI. |
| 5. Code — Format Data for GPT | code (v2) | Transform GSC JSON rows into numbered human-readable keyword text | 4. Set — Extract Fields | 6. AI Agent — Write SEO Report Email | GSC Data Collection: Fetches the top 10 keywords from Google Search Console for the past 7 days. Extracts and formats the raw API response into clean readable text for the AI. |
| 6. AI Agent — Write SEO Report Email | @n8n/n8n-nodes-langchain.agent (v1.7) | AI agent that generates a 150–200 word plain-text SEO email body | 5. Code — Format Data for GPT (main), 7. OpenAI — GPT-4o-mini Model (ai_languageModel) | 8. Set — Prepare Final Email | AI Report Writing: GPT-4o-mini reads the keyword data and writes a professional 150–200 word SEO digest email body with highlights and one actionable tip. |
| 7. OpenAI — GPT-4o-mini Model | @n8n/n8n-nodes-langchain.lmChatOpenAi (v1.2) | Language model sub-node providing GPT-4o-mini to the AI agent | — (sub-node) | 6. AI Agent — Write SEO Report Email (ai_languageModel) | AI Report Writing: GPT-4o-mini reads the keyword data and writes a professional 150–200 word SEO digest email body with highlights and one actionable tip. |
| 8. Set — Prepare Final Email | set (v3.4) | Assemble recipient, subject, body, and agency name for Gmail | 6. AI Agent — Write SEO Report Email | 9. Gmail — Send Weekly Report | Email Delivery: Assembles the subject line, recipient address, and email body into one item. Gmail sends the final report to the client. |
| 9. Gmail — Send Weekly Report | gmail (v2.1) | Send the final weekly SEO report email to the client | 8. Set — Prepare Final Email | — (terminal) | Email Delivery: Assembles the subject line, recipient address, and email body into one item. Gmail sends the final report to the client. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n and name it "Send weekly SEO keyword email reports with Google Search Console, GPT-4o-mini and Gmail".

2. **Add node: Schedule Trigger**
   - Type: `Schedule Trigger`
   - Version: 1.2
   - Configuration: Set the rule to **Cron Expression** with value `0 8 * * 1` (every Monday at 08:00).
   - No credential needed.

3. **Add node: Set — Config Values**
   - Type: `Set` (v3.4)
   - Position it to the right of the Schedule Trigger.
   - Create four string assignments:
     - `siteUrl` = `https://www.YOUR-WEBSITE.com/`
     - `clientName` = `YOUR CLIENT NAME`
     - `recipientEmail` = `user@example.com`
     - `agencyName` = `YOUR AGENCY NAME`
   - Connect: Schedule Trigger → Set — Config Values (main output → main input).

4. **Add node: HTTP Request — Fetch GSC Top Keywords**
   - Type: `HTTP Request` (v4.2)
   - Position it to the right of the Set — Config Values node.
   - Configuration:
     - **Method**: POST
     - **URL** (expression): `https://www.googleapis.com/webmasters/v3/sites/{{ encodeURIComponent($json.siteUrl) }}/searchAnalytics/query`
     - **Authentication**: OAuth2 — select or create a Google OAuth2 credential with the scope `https://www.googleapis.com/auth/webmasters.readonly`. The credential must be authorized for a Google account that has access to the target GSC property.
     - **Send Body**: Yes; **Specify Body**: JSON
     - **JSON Body** (expression):
       ```
       {
         "startDate": "{{ $now.minus({days: 7}).toFormat('yyyy-MM-dd') }}",
         "endDate": "{{ $now.minus({days: 1}).toFormat('yyyy-MM-dd') }}",
         "dimensions": ["query"],
         "rowLimit": 10,
         "orderby": [{"fieldName": "clicks", "sortOrder": "DESCENDING"}]
       }
       ```
     - **Send Headers**: Yes; add header `Content-Type` = `application/json`
   - Connect: Set — Config Values → HTTP Request (main output → main input).

5. **Add node: Set — Extract Fields**
   - Type: `Set` (v3.4)
   - Position it to the right of the HTTP Request node.
   - Create seven assignments:
     - `siteUrl` (string) = `={{ $('2. Set — Config Values').item.json.siteUrl }}`
     - `clientName` (string) = `={{ $('2. Set — Config Values').item.json.clientName }}`
     - `recipientEmail` (string) = `={{ $('2. Set — Config Values').item.json.recipientEmail }}`
     - `agencyName` (string) = `={{ $('2. Set — Config Values').item.json.agencyName }}`
     - `weekStart` (string) = `={{ $now.minus({days: 7}).toFormat('dd MMM yyyy') }}`
     - `weekEnd` (string) = `={{ $now.minus({days: 1}).toFormat('dd MMM yyyy') }}`
     - `rawRows` (string) = `={{ JSON.stringify($json.rows || []) }}`
   - Connect: HTTP Request → Set — Extract Fields (main output → main input).

6. **Add node: Code — Format Data for GPT**
   - Type: `Code` (v2)
   - Position it to the right of the Set — Extract Fields node.
   - Mode: Run Once for All Items (default)
   - JavaScript code:
     ```javascript
     const item = $input.first().json;

     let rows = [];
     try {
       rows = JSON.parse(item.rawRows || '[]');
     } catch(e) {
       rows = [];
     }

     let keywordText = '';
     if (rows.length === 0) {
       keywordText = 'No keyword data found for this week. Please check your GSC property URL and credentials.';
     } else {
       rows.forEach((row, i) => {
         const query = (row.keys && row.keys[0]) ? row.keys[0] : 'unknown';
         const clicks = row.clicks || 0;
         const impressions = row.impressions || 0;
         const ctr = ((row.ctr || 0) * 100).toFixed(1);
         const pos = (row.position || 0).toFixed(1);
         keywordText += `${i + 1}. "${query}" | Clicks: ${clicks} | Impressions: ${impressions} | CTR: ${ctr}% | Position: ${pos}\n`;
       });
     }

     return [{
       json: {
         clientName: item.clientName,
         recipientEmail: item.recipientEmail,
         agencyName: item.agencyName,
         weekStart: item.weekStart,
         weekEnd: item.weekEnd,
         keywordText: keywordText,
         totalKeywords: rows.length
       }
     }];
     ```
   - Connect: Set — Extract Fields → Code — Format Data for GPT (main output → main input).

7. **Add node: OpenAI — GPT-4o-mini Model**
   - Type: `OpenAI Chat Model` (`@n8n/n8n-nodes-langchain.lmChatOpenAi`, v1.2)
   - Position it below the AI Agent node (visually).
   - Configuration:
     - **Model**: `gpt-4o-mini` (list mode, select from dropdown)
     - **Max Tokens**: 500
     - **Temperature**: 0.6
   - Credential: Create or select an **OpenAI API** credential with a valid API key that has access to the `gpt-4o-mini` model.
   - This node has **no main input** — it will be connected as a language model sub-node to the AI Agent.

8. **Add node: AI Agent — Write SEO Report Email**
   - Type: `AI Agent` (`@n8n/n8n-nodes-langchain.agent`, v1.7)
   - Position it to the right of the Code node.
   - Configuration:
     - **Prompt Type**: Define
     - **Text** (expression — paste the full prompt):
       ```
       You are a professional SEO report writer working for {{ $json.agencyName }}.

       Your job is to write a weekly SEO performance digest email for the client.

       CLIENT NAME: {{ $json.clientName }}
       REPORT PERIOD: {{ $json.weekStart }} to {{ $json.weekEnd }}

       TOP KEYWORDS DATA (Last 7 Days from Google Search Console):
       {{ $json.keywordText }}

       WRITING INSTRUCTIONS:
       - Write the full email body only. Do NOT include subject line.
       - Use plain text only. No markdown, no asterisks, no bullet symbols, no hashtags.
       - Keep total length between 150 to 200 words.
       - Paragraph 1: Warm one-line greeting mentioning the client name and the report week.
       - Paragraph 2: Highlight the top 3 keywords by name. Mention their click count and average position. Keep it conversational.
       - Paragraph 3: One positive observation about overall performance this week.
       - Paragraph 4: One clear and actionable SEO tip the client should focus on next week.
       - Closing line: Professional sign-off. Do not write agency name — just end with 'Best regards,'
       ```
     - **Options**: None (default)
   - Connect:
     - Code — Format Data for GPT → AI Agent (main output → main input)
     - OpenAI — GPT-4o-mini Model → AI Agent (ai_languageModel output → ai_languageModel input). This is a **sub-node connection**: drag from the OpenAI model node's language model connector to the AI Agent's language model input.

9. **Add node: Set — Prepare Final Email**
   - Type: `Set` (v3.4)
   - Position it to the right of the AI Agent node.
   - Create four assignments:
     - `toEmail` (string) = `={{ $('5. Code — Format Data for GPT').item.json.recipientEmail }}`
     - `emailSubject` (string) = `=Weekly SEO Report — {{ $('5. Code — Format Data for GPT').item.json.weekStart }} to {{ $('5. Code — Format Data for GPT').item.json.weekEnd }} | {{ $('5. Code — Format Data for GPT').item.json.clientName }}`
     - `emailBody` (string) = `={{ $json.output || 'Email content could not be generated. Please check OpenAI credentials.' }}`
     - `agencyName` (string) = `={{ $('5. Code — Format Data for GPT').item.json.agencyName }}`
   - Connect: AI Agent → Set — Prepare Final Email (main output → main input).

10. **Add node: Gmail — Send Weekly Report**
    - Type: `Gmail` (v2.1)
    - Position it to the right of the Set — Prepare Final Email node.
    - Configuration:
      - **To** (expression): `={{ $json.toEmail }}`
      - **Subject** (expression): `={{ $json.emailSubject }}`
      - **Message** (expression):
        ```
        {{ $json.emailBody }}

        --
        This report was auto-generated by {{ $json.agencyName }}.
        Data source: Google Search Console | Powered by n8n + GPT-4o-mini
        ```
      - **Options**: Set `Append Attribution` to `false`
    - Credential: Create or select a **Gmail OAuth2** credential. Authorize it with the Google account that will send the emails. Ensure the Gmail API is enabled for that Google Workspace or consumer account.
    - Connect: Set — Prepare Final Email → Gmail — Send Weekly Report (main output → main input).

11. **Update configuration values** in node 2:
    - Replace `https://www.YOUR-WEBSITE.com/` with the exact GSC property URL (must match what is verified in Google Search Console, including protocol and trailing slash).
    - Replace `YOUR CLIENT NAME` with the client's business name.
    - Replace `user@example.com` with the actual recipient email address.
    - Replace `YOUR AGENCY NAME` with your agency name.

12. **Verify all three credentials** are connected and valid:
    - Google Search Console OAuth2 (in node 3) — must have `webmasters.readonly` scope.
    - OpenAI API key (in node 7) — must have access to `gpt-4o-mini`.
    - Gmail OAuth2 (in node 9) — must have `gmail.send` or equivalent scope.

13. **Activate the workflow** using the toggle in the top-right corner. It will execute automatically every Monday at 08:00.

14. **Optional — Test manually:** Click "Execute Workflow" or "Execute Node" on the Schedule Trigger to run it immediately and verify the end-to-end flow before the first automated run.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Google Search Console API documentation — Search Analytics query endpoint reference | https://developers.google.com/webmaster-tools/search-console-api-original/v3/searchanalytics/query |
| OpenAI GPT-4o-mini model pricing and capabilities | https://platform.openai.com/docs/models/gpt-4o-mini |
| Gmail API send limits — 500/day for consumer Gmail, 2,000/day for Workspace | https://support.google.com/a/answer/166852 |
| n8n AI Agent node documentation | https://docs.n8n.io/integrations/builtin/cluster-nodes/root-nodes/n8n-nodes-langchain.agent/ |
| n8n OpenAI Chat Model node documentation | https://docs.n8n.io/integrations/builtin/cluster-nodes/sub-nodes/n8n-nodes-langchain.lmchatopenai/ |
| The `siteUrl` value must exactly match the GSC property URL including `https://`, `www.` prefix (if used), and trailing `/` — any mismatch results in a 403 error from the API. | Setup note for node 2 and node 3 |
| The GSC API `ctr` field is returned as a decimal fraction (e.g., 0.045 means 4.5%). The Code node multiplies by 100 for human-readable output. | Implementation detail for node 5 |
| The AI Agent's output field is `output`. If the model or agent structure changes in a future n8n version, the field name may differ — node 8 includes a fallback message. | Compatibility note for node 8 |
| Setting `appendAttribution: false` on the Gmail node prevents n8n from adding its own "Sent via n8n" footer line to the email. | Configuration note for node 9 |
| The cron expression `0 8 * * 1` runs in the n8n server's timezone. Verify the server timezone matches your intended delivery timezone. | Deployment consideration |
| This workflow sends one email per execution. For multiple clients, duplicate the workflow or extend it with an Items list and a loop, each item containing a different client configuration. | Scaling suggestion |