Analyze ad performance from Meta, Google and Microsoft using Groq Llama 3.3 and Gmail

https://n8nworkflows.xyz/workflows/analyze-ad-performance-from-meta--google-and-microsoft-using-groq-llama-3-3-and-gmail-14011


# Analyze ad performance from Meta, Google and Microsoft using Groq Llama 3.3 and Gmail

# 1. Workflow Overview

This workflow generates a monthly ad performance report across **Meta Ads, Google Ads, and Microsoft Ads**, uses **Groq Llama 3.3 70B** to produce expert commentary, converts the results into a styled **HTML email**, and sends that report through **Gmail**.

Typical use cases:
- Monthly executive reporting for paid media accounts
- Cross-platform performance comparison
- Automated AI-assisted insights for marketing teams
- Reusable reporting base for agencies or in-house growth teams

The workflow is organized into the following logical blocks:

## 1.1 Scheduled Trigger
The workflow starts automatically on a monthly schedule.

## 1.2 Data Extraction from Google Sheets
It reads campaign export data from three tabs in a single Google Sheet:
- Google
- Meta
- Microsoft

## 1.3 Data Consolidation and KPI Preparation
The rows from all three sources are merged, normalized into a shared structure, aggregated into KPIs, and transformed into a structured prompt for the AI model.

## 1.4 AI Insight Generation with Groq
The prepared prompt is sent to the Groq Chat Completions API using the `llama-3.3-70b-versatile` model.

## 1.5 Merge KPI Data with AI Output
The workflow combines:
- the KPI object generated in the Code node
- the Groq response payload

This merged object becomes the source for the final report.

## 1.6 HTML Report Composition and Email Delivery
A second Code node creates a complete HTML report with charts, summary tables, campaign tables, and formatted AI commentary. The final report is emailed via Gmail.

---

# 2. Block-by-Block Analysis

## 2.1 Scheduled Trigger

**Overview:**  
This block defines the workflow entry point. It launches the workflow automatically on a monthly cadence and fans out to the three Google Sheets reader nodes in parallel.

**Nodes Involved:**  
- Trigger - Monthly Schedule

### Node Details

#### Trigger - Monthly Schedule
- **Type and technical role:** `n8n-nodes-base.scheduleTrigger`  
  Entry-point trigger node that starts the workflow automatically based on a recurring schedule.
- **Configuration choices:**  
  Configured with a monthly interval rule. The JSON indicates a monthly schedule with `triggerAtHour: 9`. The accompanying note says it runs on the 1st day of each month, though the exact day should be verified in the n8n UI because the rule structure is abbreviated in export form.
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  - No input; this is a trigger node.
  - Outputs to:
    - Read - Google ads
    - Read - Meta ads
    - Read - Microsoft ads
- **Version-specific requirements:**  
  `typeVersion: 1.3`
- **Edge cases or potential failure types:**  
  - Misconfigured timezone at the workflow or instance level
  - Trigger not firing if workflow is inactive
  - Ambiguity in “first day of month” if not explicitly set in UI after import
- **Sub-workflow reference:**  
  None.

---

## 2.2 Data Extraction from Google Sheets

**Overview:**  
This block reads campaign data from three platform-specific tabs in a single Google Sheets document. Each node pulls rows from its respective tab and sends them to a merge node.

**Nodes Involved:**  
- Read - Google ads
- Read - Meta ads
- Read - Microsoft ads
- Merge all platform

### Node Details

#### Read - Google ads
- **Type and technical role:** `n8n-nodes-base.googleSheets`  
  Reads rows from the Google Ads tab.
- **Configuration choices:**  
  - Uses a specific Google Sheet document named **Campaign data**
  - Reads from the tab named **Google**
  - No special read options are set
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  - Input from: Trigger - Monthly Schedule
  - Output to: Merge all platform (input 0)
- **Version-specific requirements:**  
  `typeVersion: 4.7`
- **Edge cases or potential failure types:**  
  - Google OAuth credential missing or expired
  - Document or tab not shared with the credentialed account
  - Header row mismatch causing later normalization issues
  - Empty tab leading to zero items
- **Sub-workflow reference:**  
  None.

#### Read - Meta ads
- **Type and technical role:** `n8n-nodes-base.googleSheets`  
  Reads rows from the Meta Ads tab.
- **Configuration choices:**  
  - Uses the same Google Sheet document
  - Reads from the tab named **Meta**
  - No special read options are set
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  - Input from: Trigger - Monthly Schedule
  - Output to: Merge all platform (input 1)
- **Version-specific requirements:**  
  `typeVersion: 4.7`
- **Edge cases or potential failure types:**  
  - OAuth or permission problems
  - Missing expected Meta export columns such as `Campaign name`, `Amount spent (USD)`, `Results`, `Purchase ROAS (return on ad spend)`
  - Empty tab
- **Sub-workflow reference:**  
  None.

#### Read - Microsoft ads
- **Type and technical role:** `n8n-nodes-base.googleSheets`  
  Reads rows from the Microsoft Ads tab.
- **Configuration choices:**  
  - Uses the same Google Sheet document
  - Reads from the tab named **Microsoft**
  - No special read options are set
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  - Input from: Trigger - Monthly Schedule
  - Output to: Merge all platform (input 2)
- **Version-specific requirements:**  
  `typeVersion: 4.7`
- **Edge cases or potential failure types:**  
  - OAuth or sharing issues
  - Missing expected Microsoft fields such as `Status`, `Impr.`, `Spend`, `Conv.`
  - Empty tab
- **Sub-workflow reference:**  
  None.

#### Merge all platform
- **Type and technical role:** `n8n-nodes-base.merge`  
  Collects rows from three inputs into a combined stream for downstream processing.
- **Configuration choices:**  
  - `numberInputs: 3`
  - Used as a three-input consolidation point for Google, Meta, and Microsoft data
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  - Inputs from:
    - Read - Google ads
    - Read - Meta ads
    - Read - Microsoft ads
  - Output to:
    - Calculate - KPIs & Build Prompt
- **Version-specific requirements:**  
  `typeVersion: 3.2`
- **Edge cases or potential failure types:**  
  - If one source returns no items, downstream output shape depends on actual Merge node behavior in current n8n version and selected mode
  - Inconsistent row structure is expected and handled later, but completely missing items from one input can affect platform coverage
- **Sub-workflow reference:**  
  None.

---

## 2.3 Data Consolidation and KPI Preparation

**Overview:**  
This block is the analytical core of the workflow. It standardizes the three platform-specific row schemas, calculates aggregate KPIs, identifies notable campaign flags, and builds a strongly structured prompt for the Groq model.

**Nodes Involved:**  
- Calculate - KPIs & Build Prompt

### Node Details

#### Calculate - KPIs & Build Prompt
- **Type and technical role:** `n8n-nodes-base.code`  
  JavaScript transformation node that normalizes rows, computes metrics, and generates the AI prompt.
- **Configuration choices:**  
  The script:
  1. Parses numeric values robustly with a `num()` helper
  2. Detects source platform using column signatures
  3. Normalizes each row into a shared object model
  4. Aggregates metrics by platform and globally
  5. Detects best/worst campaign patterns
  6. Builds a natural-language prompt for Groq
- **Key expressions or variables used:**  
  Major internal variables include:
  - `items.map(i => i.json)` to read merged rows
  - `detectPlatform(r)` based on headers like:
    - `Campaign status` → Google
    - `Campaign name` → Meta
    - `Status` + `Impr.` → Microsoft
  - `normalise(r)` returning:
    - `platform`
    - `campaign`
    - `status`
    - `type`
    - `impressions`
    - `clicks`
    - `spend`
    - `conversions`
    - `roas`
    - `cpm`
    - `purchases`
    - `budget`
  - Aggregated KPI objects:
    - `tG`
    - `tM`
    - `tMeta`
    - `tAll`
  - Insight helpers:
    - `bestMS`
    - `worstMeta`
    - `bestMeta`
    - `budgetPaused`
  - Final prompt field:
    - `groqPrompt`
- **Input and output connections:**  
  - Input from: Merge all platform
  - Outputs to:
    - Groq - Generate Insights
    - Merge - KPI + AI
- **Version-specific requirements:**  
  `typeVersion: 2`
- **Edge cases or potential failure types:**  
  - Column names must match exactly or platform detection may fail
  - Numeric coercion may silently convert malformed values to `0`
  - Google campaigns are always flagged in prompt as “0 conversions across all campaigns,” regardless of whether that remains universally true in future data exports
  - Meta clicks are hardcoded to `0` because source mapping does not extract a click field
  - Microsoft `roas` and `cpm` are set to `0`
  - Rows with unrecognized schema are discarded
  - `meta` only includes rows with `spend > 0`
  - `microsoft` is fully stored, but total calculations use only spend-positive records in some places
  - Prompt month and brand are hardcoded:
    - **February 2026**
    - **Biogetica**
    If reused later, these should be made dynamic
- **Sub-workflow reference:**  
  None.

**Output produced by this node:**
- `json.kpi`: aggregated metrics and campaign lists
- `json.groqPrompt`: AI-ready prompt text

---

## 2.4 AI Insight Generation with Groq

**Overview:**  
This block sends the structured prompt to Groq’s OpenAI-compatible Chat Completions API and receives narrative analysis in response.

**Nodes Involved:**  
- Groq - Generate Insights

### Node Details

#### Groq - Generate Insights
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Makes an authenticated POST request to Groq’s chat completions endpoint.
- **Configuration choices:**  
  - Method: `POST`
  - URL: `https://api.groq.com/openai/v1/chat/completions`
  - Authentication: Generic credential type using HTTP Header Auth
  - Header explicitly sets `Content-Type: application/json`
  - JSON body is dynamically built as:
    - `model: "llama-3.3-70b-versatile"`
    - `messages: [{ role: "user", content: $json.groqPrompt }]`
- **Key expressions or variables used:**  
  - `{{$json.groqPrompt}}` inside the request body expression
- **Input and output connections:**  
  - Input from: Calculate - KPIs & Build Prompt
  - Output to: Merge - KPI + AI
- **Version-specific requirements:**  
  `typeVersion: 4.4`
- **Edge cases or potential failure types:**  
  - Invalid or missing Groq API key in Header Auth credential
  - 401/403 authentication errors
  - Rate limiting or quota exhaustion
  - Timeout or transient API failure
  - Unexpected response schema breaking downstream `choices?.[0]?.message?.content` lookup
  - Prompt length growth if spreadsheet becomes very large
- **Sub-workflow reference:**  
  None.

---

## 2.5 Merge KPI Data with AI Output

**Overview:**  
This block combines the KPI dataset from the first Code node with the AI response from Groq. The output is designed so the next Code node can access both in a single item.

**Nodes Involved:**  
- Merge - KPI + AI

### Node Details

#### Merge - KPI + AI
- **Type and technical role:** `n8n-nodes-base.merge`  
  Combines two inputs by item position.
- **Configuration choices:**  
  - Mode: `combine`
  - Combine by: `combineByPosition`
  - This assumes the KPI branch and Groq branch each output one corresponding item
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  - Inputs from:
    - Groq - Generate Insights
    - Calculate - KPIs & Build Prompt
  - Output to:
    - Build - HTML Report
- **Version-specific requirements:**  
  `typeVersion: 3.2`
- **Edge cases or potential failure types:**  
  - If one branch returns zero items, the merge may produce no usable output
  - If branch item counts differ, pairing by position may misalign data
  - If Groq returns an unexpected object shape, downstream AI extraction may fail gracefully to empty string
- **Sub-workflow reference:**  
  None.

---

## 2.6 HTML Report Composition and Email Delivery

**Overview:**  
This block transforms KPI data and AI commentary into a polished HTML report, generates the email subject line, and sends the final message through Gmail.

**Nodes Involved:**  
- Build - HTML Report
- Send a message

### Node Details

#### Build - HTML Report
- **Type and technical role:** `n8n-nodes-base.code`  
  Generates the final HTML email and subject line.
- **Configuration choices:**  
  The script:
  - Reads combined data from `items[0].json`
  - Extracts:
    - `kpi`
    - `choices[0].message.content` from Groq response
  - Builds:
    - Spend donut chart in inline SVG
    - Conversion bar chart in inline SVG
    - Platform summary table
    - Detailed Meta, Microsoft, and Google campaign tables
    - Formatted AI analysis section
    - Footer and subject line
- **Key expressions or variables used:**  
  Important internal fields:
  - `const j = items[0].json`
  - `const aiRaw = j.choices?.[0]?.message?.content || ''`
  - Helpers:
    - `usd()`
    - `fmt()`
    - `donut()`
    - `convBars()`
    - `formatAI()`
  - Returned output:
    - `html`
    - `subject`
- **Input and output connections:**  
  - Input from: Merge - KPI + AI
  - Output to: Send a message
- **Version-specific requirements:**  
  `typeVersion: 2`
- **Edge cases or potential failure types:**  
  - Assumes merged payload is accessible via `items[0].json`
  - If AI text is missing, node falls back to “AI insights not available for this run.”
  - Several report labels are hardcoded:
    - “Monthly Performance Report · February 2026”
    - Google subtitle: “All campaigns returned 0 conversions — verify pixel setup”
  - Microsoft campaign rows attempt to show `r.ctr`, but the normalization script does not compute per-row CTR, so that table column may display blank or `—`
  - Status dot only marks exact `active` or `Enabled` as green; other valid active labels may appear gray
  - Long campaign names are truncated
  - HTML email rendering can vary between email clients
- **Sub-workflow reference:**  
  None.

#### Send a message
- **Type and technical role:** `n8n-nodes-base.gmail`  
  Sends the generated HTML report email through Gmail.
- **Configuration choices:**  
  - Recipient: `user@example.com`
  - Subject: dynamic from `{{$json.subject}}`
  - Message body: dynamic from `{{$json.html}}`
  - Uses Gmail OAuth2 credentials
- **Key expressions or variables used:**  
  - `{{$json.subject}}`
  - `{{$json.html}}`
- **Input and output connections:**  
  - Input from: Build - HTML Report
  - No downstream node
- **Version-specific requirements:**  
  `typeVersion: 2.2`
- **Edge cases or potential failure types:**  
  - Invalid Gmail OAuth2 credential
  - Gmail sending limits
  - HTML body may render differently than intended
  - Recipient placeholder must be replaced before production use
- **Sub-workflow reference:**  
  None.

---

## 2.7 Documentation / Sticky Notes Layer

**Overview:**  
These nodes do not affect execution. They provide setup guidance, workflow purpose, and block-specific comments inside the canvas.

**Nodes Involved:**  
- Sticky Note
- Sticky Note1
- Sticky Note2
- Sticky Note3

### Node Details

#### Sticky Note
- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Visual documentation for the data preparation area.
- **Configuration choices:**  
  Describes reading campaign data from Google Sheets and formatting it for AI analysis.
- **Input and output connections:**  
  None.
- **Version-specific requirements:**  
  `typeVersion: 1`
- **Edge cases or potential failure types:**  
  None; non-executing node.
- **Sub-workflow reference:**  
  None.

#### Sticky Note1
- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Visual documentation for the schedule trigger area.
- **Configuration choices:**  
  Explains that the workflow runs monthly and that schedule can be changed in trigger settings.
- **Input and output connections:**  
  None.
- **Version-specific requirements:**  
  `typeVersion: 1`
- **Edge cases or potential failure types:**  
  None.
- **Sub-workflow reference:**  
  None.

#### Sticky Note2
- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Visual documentation for the AI and email section.
- **Configuration choices:**  
  Notes Groq and Gmail setup requirements.
- **Input and output connections:**  
  None.
- **Version-specific requirements:**  
  `typeVersion: 1`
- **Edge cases or potential failure types:**  
  None.
- **Sub-workflow reference:**  
  None.

#### Sticky Note3
- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Full workflow explanation and setup checklist.
- **Configuration choices:**  
  Summarizes workflow behavior and setup sequence.
- **Input and output connections:**  
  None.
- **Version-specific requirements:**  
  `typeVersion: 1`
- **Edge cases or potential failure types:**  
  None.
- **Sub-workflow reference:**  
  None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Trigger - Monthly Schedule | Schedule Trigger | Monthly workflow entry point |  | Read - Google ads; Read - Meta ads; Read - Microsoft ads | ## TRIGGER<br>Runs every Months 1st day automatically.<br>Change schedule in the trigger node settings. |
| Read - Google ads | Google Sheets | Read Google Ads tab rows from spreadsheet | Trigger - Monthly Schedule | Merge all platform | ## DATA FETCH & FORMAT<br>Reads campaign data from Google Sheet.<br>Formats rows into a single text string for AI analysis.<br>Setup: Add your Sheet ID in the Google Sheets node. |
| Read - Meta ads | Google Sheets | Read Meta Ads tab rows from spreadsheet | Trigger - Monthly Schedule | Merge all platform | ## DATA FETCH & FORMAT<br>Reads campaign data from Google Sheet.<br>Formats rows into a single text string for AI analysis.<br>Setup: Add your Sheet ID in the Google Sheets node. |
| Read - Microsoft ads | Google Sheets | Read Microsoft Ads tab rows from spreadsheet | Trigger - Monthly Schedule | Merge all platform | ## DATA FETCH & FORMAT<br>Reads campaign data from Google Sheet.<br>Formats rows into a single text string for AI analysis.<br>Setup: Add your Sheet ID in the Google Sheets node. |
| Merge all platform | Merge | Combine platform row streams before KPI calculation | Read - Google ads; Read - Meta ads; Read - Microsoft ads | Calculate - KPIs & Build Prompt | ## DATA FETCH & FORMAT<br>Reads campaign data from Google Sheet.<br>Formats rows into a single text string for AI analysis.<br>Setup: Add your Sheet ID in the Google Sheets node. |
| Calculate - KPIs & Build Prompt | Code | Normalize data, compute KPIs, build Groq prompt | Merge all platform | Groq - Generate Insights; Merge - KPI + AI |  |
| Groq - Generate Insights | HTTP Request | Send prompt to Groq chat completions API | Calculate - KPIs & Build Prompt | Merge - KPI + AI | ## AI ANALYSIS & EMAIL<br>Sends campaign data to Groq (Llama 3.3) for analysis.<br>Extracts the report and emails it automatically.<br>Setup: Add your Groq API key and Gmail credentials. |
| Merge - KPI + AI | Merge | Combine KPI output with AI response by position | Groq - Generate Insights; Calculate - KPIs & Build Prompt | Build - HTML Report | ## AI ANALYSIS & EMAIL<br>Sends campaign data to Groq (Llama 3.3) for analysis.<br>Extracts the report and emails it automatically.<br>Setup: Add your Groq API key and Gmail credentials. |
| Build - HTML Report | Code | Build final HTML report and email subject | Merge - KPI + AI | Send a message | ## AI ANALYSIS & EMAIL<br>Sends campaign data to Groq (Llama 3.3) for analysis.<br>Extracts the report and emails it automatically.<br>Setup: Add your Groq API key and Gmail credentials. |
| Send a message | Gmail | Send report email | Build - HTML Report |  | ## AI ANALYSIS & EMAIL<br>Sends campaign data to Groq (Llama 3.3) for analysis.<br>Extracts the report and emails it automatically.<br>Setup: Add your Groq API key and Gmail credentials. |
| Sticky Note | Sticky Note | Canvas documentation for data-fetch block |  |  |  |
| Sticky Note1 | Sticky Note | Canvas documentation for trigger block |  |  |  |
| Sticky Note2 | Sticky Note | Canvas documentation for AI/email block |  |  |  |
| Sticky Note3 | Sticky Note | Canvas documentation with overall setup instructions |  |  |  |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it something like: **Analyze ad performance across Meta, Google and Microsoft with AI**
   - Leave it inactive until all credentials and tests are complete.

2. **Add the Schedule Trigger node**
   - Node type: **Schedule Trigger**
   - Name: `Trigger - Monthly Schedule`
   - Configure it to run **monthly**, ideally on the **1st day**, at **09:00**
   - Confirm the workflow timezone matches your reporting expectations

3. **Prepare the source spreadsheet**
   - Create one Google Sheets file, e.g. **Campaign data**
   - Create three tabs:
     - `Google`
     - `Meta`
     - `Microsoft`
   - Paste each platform’s monthly export into the matching tab
   - Keep header names consistent with the workflow logic

4. **Add the Google Ads reader**
   - Node type: **Google Sheets**
   - Name: `Read - Google ads`
   - Operation: read rows from sheet
   - Select the spreadsheet document
   - Select the `Google` tab
   - Add **Google Sheets OAuth2 API** credentials
   - Connect:
     - `Trigger - Monthly Schedule` → `Read - Google ads`

5. **Add the Meta Ads reader**
   - Node type: **Google Sheets**
   - Name: `Read - Meta ads`
   - Select the same spreadsheet document
   - Select the `Meta` tab
   - Use the same Google Sheets OAuth2 credential or another authorized account
   - Connect:
     - `Trigger - Monthly Schedule` → `Read - Meta ads`

6. **Add the Microsoft Ads reader**
   - Node type: **Google Sheets**
   - Name: `Read - Microsoft ads`
   - Select the same spreadsheet document
   - Select the `Microsoft` tab
   - Use valid Google Sheets OAuth2 credentials
   - Connect:
     - `Trigger - Monthly Schedule` → `Read - Microsoft ads`

7. **Add the first Merge node**
   - Node type: **Merge**
   - Name: `Merge all platform`
   - Set **Number of Inputs** to `3`
   - Connect:
     - `Read - Google ads` → input 0
     - `Read - Meta ads` → input 1
     - `Read - Microsoft ads` → input 2

8. **Add the KPI and prompt Code node**
   - Node type: **Code**
   - Name: `Calculate - KPIs & Build Prompt`
   - Language: JavaScript
   - Paste logic that:
     - detects platform by header pattern
     - normalizes rows into a common schema
     - computes platform and global aggregates
     - identifies key flags like best/worst campaigns
     - builds a Groq prompt with explicit reporting sections
   - Connect:
     - `Merge all platform` → `Calculate - KPIs & Build Prompt`

9. **Ensure your input headers match the code expectations**
   - Google expected headers include at least:
     - `Campaign`
     - `Campaign status`
     - `Campaign type`
     - `Impr.`
     - `Clicks`
     - `Cost`
     - `Conversions`
     - `Conv. value / cost`
     - `Budget`
   - Meta expected headers include at least:
     - `Campaign name`
     - `Campaign delivery`
     - `Impressions`
     - `Amount spent (USD)`
     - `Results`
     - `Purchase ROAS (return on ad spend)`
     - `CPM (cost per 1,000 impressions) (USD)`
     - `Purchases`
     - `Ad set budget`
   - Microsoft expected headers include at least:
     - `Campaign`
     - `Status`
     - `Campaign Type`
     - `Impr.`
     - `Clicks`
     - `Spend`
     - `Conv.`
     - `Budget`

10. **Add the Groq HTTP Request node**
    - Node type: **HTTP Request**
    - Name: `Groq - Generate Insights`
    - Method: `POST`
    - URL: `https://api.groq.com/openai/v1/chat/completions`
    - Authentication: **Generic Credential Type**
    - Generic auth type: **HTTP Header Auth**
    - Add header:
      - `Content-Type: application/json`
    - Enable **Send Body**
    - Body type: **JSON**
    - Use this body structure conceptually:
      - `model = llama-3.3-70b-versatile`
      - `messages = [{ role: "user", content: $json.groqPrompt }]`
    - Create Header Auth credential with Groq API key, typically:
      - Header name: `Authorization`
      - Header value: `Bearer YOUR_GROQ_API_KEY`
    - Connect:
      - `Calculate - KPIs & Build Prompt` → `Groq - Generate Insights`

11. **Add the second Merge node**
    - Node type: **Merge**
    - Name: `Merge - KPI + AI`
    - Mode: **Combine**
    - Combine By: **Position**
    - Connect:
      - `Groq - Generate Insights` → input 0
      - `Calculate - KPIs & Build Prompt` → input 1

12. **Add the HTML builder Code node**
    - Node type: **Code**
    - Name: `Build - HTML Report`
    - Language: JavaScript
    - Paste logic that:
      - reads merged item data from `items[0].json`
      - extracts `kpi`
      - extracts AI text from `choices[0].message.content`
      - builds:
        - KPI strip
        - spend donut SVG
        - conversions bar SVG
        - platform summary table
        - Meta campaign table
        - Microsoft campaign table
        - Google campaign table
        - AI analysis section
      - returns:
        - `html`
        - `subject`
    - Connect:
      - `Merge - KPI + AI` → `Build - HTML Report`

13. **Add the Gmail send node**
    - Node type: **Gmail**
    - Name: `Send a message`
    - Operation: send message
    - Configure:
      - To: your real recipient address, replacing `user@example.com`
      - Subject: `{{$json.subject}}`
      - Message: `{{$json.html}}`
    - Add **Gmail OAuth2 API** credentials
    - Connect:
      - `Build - HTML Report` → `Send a message`

14. **Optionally add sticky notes**
    - Add a note near the trigger with text about monthly execution
    - Add a note near the Google Sheets and merge block describing sheet setup
    - Add a note near Groq/Gmail explaining required credentials
    - Add an overall note describing workflow purpose and setup steps

15. **Test the source data first**
    - Run each Google Sheets node individually
    - Confirm row output exists
    - Confirm headers match expected field names exactly

16. **Test KPI generation**
    - Run `Merge all platform`
    - Run `Calculate - KPIs & Build Prompt`
    - Inspect output for:
      - `kpi`
      - `groqPrompt`

17. **Test Groq API call**
    - Run `Groq - Generate Insights`
    - Confirm the response includes:
      - `choices`
      - `choices[0].message.content`

18. **Test final merge and HTML**
    - Run `Merge - KPI + AI`
    - Run `Build - HTML Report`
    - Confirm output contains:
      - `html`
      - `subject`

19. **Test email delivery**
    - Run `Send a message`
    - Verify:
      - email subject formatting
      - HTML layout in Gmail or target client
      - SVG/chart rendering
      - AI section readability

20. **Activate the workflow**
    - Only after all credentials, recipients, and source tabs are confirmed
    - Keep monitoring the first scheduled run

## Credential configuration summary

### Google Sheets OAuth2 API
- Required by:
  - Read - Google ads
  - Read - Meta ads
  - Read - Microsoft ads
- Must have access to the spreadsheet file

### Groq API via HTTP Header Auth
- Required by:
  - Groq - Generate Insights
- Typical configuration:
  - Header: `Authorization`
  - Value: `Bearer <GROQ_API_KEY>`

### Gmail OAuth2 API
- Required by:
  - Send a message
- Must allow send-mail access for the connected Gmail account

## Important build constraints

- The workflow assumes **one merged KPI item** and **one Groq response item**
- `Merge - KPI + AI` relies on **Combine by Position**
- The HTML builder assumes the Groq response uses the OpenAI-style structure:
  - `choices[0].message.content`
- Month and branding are currently hardcoded in the code:
  - `February 2026`
  - `March 2026`
  - `Biogetica`
- If you want reusable monthly automation, make these dynamic using:
  - expressions from current date
  - a Set node
  - a pre-processing Code node

## No sub-workflows
This workflow does **not** invoke any sub-workflow or Execute Workflow node.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| This workflow automatically generates a detailed AI-powered campaign performance report across Meta, Google and Microsoft Ads and emails it to your team every month. | Overall workflow purpose |
| It reads campaign data from three tabs in a Google Sheet (Google, Meta, Microsoft), merges all rows, and passes them to a Code node that calculates KPIs and builds a structured prompt. Groq AI (Llama 3.3 70B) then analyzes the data and generates expert insights. A second Code node combines the KPIs and AI analysis into a full HTML email with platform tables, charts, benchmarks and recommendations — sent automatically via Gmail. | Overall workflow behavior |
| Setup steps: 1. Create a Google Sheet with 3 tabs: Google, Meta, Microsoft. 2. Paste your monthly ad exports into the matching tab. 3. Connect your Google account in the 3 Sheets nodes and select the correct tab in each. 4. Add your Groq API key in the HTTP Request node header. 5. Connect your Gmail account in the Send node and set your recipient email. 6. Activate — the workflow runs automatically on schedule. | Setup guidance |
| The trigger note states the workflow runs every month on the 1st day automatically. Verify the exact recurrence rule after import in the n8n UI. | Operational caution |
| Several strings in the code are hardcoded to February/March 2026 and to the brand name Biogetica. These should be parameterized for long-term production use. | Maintenance note |
| The report logic uses spreadsheet exports rather than direct platform APIs. This reduces API integration complexity, but makes the workflow dependent on stable spreadsheet column names and manual/export-based data freshness. | Architecture note |