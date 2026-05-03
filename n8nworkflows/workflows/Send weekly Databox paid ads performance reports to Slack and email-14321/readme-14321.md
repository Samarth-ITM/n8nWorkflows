Send weekly Databox paid ads performance reports to Slack and email

https://n8nworkflows.xyz/workflows/send-weekly-databox-paid-ads-performance-reports-to-slack-and-email-14321


# Send weekly Databox paid ads performance reports to Slack and email

# 1. Workflow Overview

This workflow automates a weekly paid advertising performance report using Databox as the data source, an AI Agent to discover and aggregate platform data, and Slack/Gmail to distribute the final report.

Its core purpose is to eliminate manual weekly reporting by:
- running on a schedule every Monday at 9:00 AM,
- discovering all connected paid ad platforms inside Databox,
- pulling six core metrics for the last 7 days and previous 7 days,
- calculating week-over-week changes,
- formatting two deliverables:
  - a concise Slack summary,
  - a richer HTML email report.

The workflow has a single entry point and no sub-workflows. It relies on one AI tool integration (Databox MCP) and one language model integration (OpenAI Chat Model).

## 1.1 Scheduled Trigger and Date Context

This block starts the workflow every Monday at 9 AM and provides a current-date value that the AI Agent uses as the reporting anchor.

## 1.2 AI-Led Databox Discovery and Report Generation

This block connects the AI Agent to:
- the OpenAI Chat Model for reasoning and formatting,
- the Databox MCP Tool for account/data source/metric retrieval.

The agent is instructed to discover connected paid ad platforms, fetch platform metrics, compute comparisons, and return a dual-format output separated by a delimiter.

## 1.3 Output Parsing and Delivery

This block parses the agent’s combined output into:
- Slack text,
- email subject,
- HTML email body,

then sends both outputs in parallel.

---

# 2. Block-by-Block Analysis

## 2.1 Scheduled Trigger and Date Context

### Overview
This block initiates the workflow on a recurring weekly schedule and supplies a current date value for downstream prompt logic. The agent uses this date to infer “last 7 days” and “previous 7 days.”

### Nodes Involved
- Every Monday 9 AM
- Get Current Date

### Node Details

#### Every Monday 9 AM
- **Type and technical role:** `n8n-nodes-base.scheduleTrigger`  
  Entry point that runs the workflow automatically on a cron schedule.
- **Configuration choices:**
  - Uses a cron expression: `0 9 * * 1`
  - This means every Monday at 09:00 server/workflow timezone.
- **Key expressions or variables used:** None.
- **Input and output connections:**
  - No input; it is the trigger.
  - Outputs to **Get Current Date**.
- **Version-specific requirements:**
  - Type version `1.2`.
  - Behavior may depend on the n8n instance timezone setting.
- **Edge cases or potential failure types:**
  - Unexpected execution time if the instance timezone differs from business expectations.
  - Workflow remains inactive until manually activated.
- **Sub-workflow reference:** None.

#### Get Current Date
- **Type and technical role:** `n8n-nodes-base.dateTime`  
  Produces a formatted current date value for prompt injection.
- **Configuration choices:**
  - No custom options are set.
  - The downstream prompt expects `{{ $json.formattedDate }}`, so this node is relied on to emit a `formattedDate` field.
- **Key expressions or variables used:**
  - Downstream usage: `{{ $json.formattedDate }}`
- **Input and output connections:**
  - Input from **Every Monday 9 AM**
  - Output to **Paid Ads Reporting Agent**
- **Version-specific requirements:**
  - Type version `2`.
  - Output field naming should be verified in the target n8n version, especially that `formattedDate` exists as expected.
- **Edge cases or potential failure types:**
  - If the node output schema differs from expected, the agent prompt could receive an empty or invalid date.
  - Locale/timezone formatting may affect the agent’s interpretation of “today.”
- **Sub-workflow reference:** None.

---

## 2.2 AI-Led Databox Discovery and Report Generation

### Overview
This block is the functional core of the workflow. The AI Agent receives today’s date, uses the Databox MCP Tool to discover connected paid platforms and pull weekly metric data, then returns a Slack section and an HTML email section in one structured response.

### Nodes Involved
- Paid Ads Reporting Agent
- OpenAI Chat Model
- Databox MCP Tool

### Node Details

#### Paid Ads Reporting Agent
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent`  
  An LLM-driven orchestration node that combines prompt instructions, a language model, and an MCP tool for external data retrieval.
- **Configuration choices:**
  - Prompt type is defined directly in the node.
  - Main prompt injects today’s date via:
    - `Today's Date: {{ $json.formattedDate }}`
  - The prompt instructs the agent to:
    - determine current and previous 7-day ranges,
    - list Databox accounts,
    - discover connected paid ad data sources,
    - fetch six metrics per platform,
    - compute week-over-week changes,
    - aggregate totals and weighted metrics,
    - output exactly two sections separated by `---SEPARATOR---`.
  - System message imposes strict formatting and behavior requirements:
    - skip disconnected/no-data platforms silently,
    - return fallback messaging if no paid activity exists,
    - color-code HTML output,
    - avoid exposing intermediate calculations,
    - format numbers/currency/percentages consistently.
  - `alwaysOutputData` is enabled.
  - `retryOnFail` is disabled.
- **Key expressions or variables used:**
  - `{{ $json.formattedDate }}`
  - Required output delimiter: `---SEPARATOR---`
- **Input and output connections:**
  - Main input from **Get Current Date**
  - AI language model input from **OpenAI Chat Model**
  - AI tool input from **Databox MCP Tool**
  - Main output to **Parse AI Output**
- **Version-specific requirements:**
  - Type version `3`
  - Requires n8n setup that supports LangChain Agent nodes and MCP tool wiring.
- **Edge cases or potential failure types:**
  - LLM may ignore output format instructions and omit the separator.
  - LLM may generate malformed HTML.
  - Databox tool calls may fail due to auth or unavailable metrics.
  - Some platforms may use nonstandard metric names, causing partial retrieval.
  - Week-over-week calculation ambiguity if prior values are zero.
  - Prompt assumes the tool exposes functions such as `list_accounts`, `list_data_sources`, and `load_metric_data`; if the MCP server changes tool names, the workflow may break logically.
- **Sub-workflow reference:** None.

#### OpenAI Chat Model
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
  Provides the LLM used by the agent for reasoning, tool selection, calculations, and formatting.
- **Configuration choices:**
  - Model selected: `gpt-5-mini`
  - No special options or built-in tools configured.
- **Key expressions or variables used:** None directly.
- **Input and output connections:**
  - Outputs an AI language model connection to **Paid Ads Reporting Agent**
- **Version-specific requirements:**
  - Type version `1.3`
  - Requires valid OpenAI credentials and model availability in the connected account.
- **Edge cases or potential failure types:**
  - Invalid API key or insufficient quota.
  - Model name not available in a given OpenAI account/region.
  - Token/context limitations if too many platforms/metrics are returned.
  - Model behavior variance may affect formatting reliability.
- **Sub-workflow reference:** None.

#### Databox MCP Tool
- **Type and technical role:** `@n8n/n8n-nodes-langchain.mcpClientTool`  
  Exposes Databox MCP server capabilities to the AI Agent as callable tools.
- **Configuration choices:**
  - Endpoint URL: `https://mcp.databox.com/mcp`
  - Authentication: `mcpOAuth2Api`
  - No advanced options configured.
- **Key expressions or variables used:** None directly in node config.
- **Input and output connections:**
  - Outputs an AI tool connection to **Paid Ads Reporting Agent**
- **Version-specific requirements:**
  - Type version `1.2`
  - Requires MCP-compatible n8n environment and OAuth2 credential setup for Databox.
- **Edge cases or potential failure types:**
  - OAuth authorization failure or expired token.
  - MCP endpoint downtime/network failure.
  - Tool schema changes on the Databox side.
  - Account has no connected paid ad platforms.
  - Metrics may not exist or may have different names for a given source.
- **Sub-workflow reference:** None.

---

## 2.3 Output Parsing and Delivery

### Overview
This block validates and splits the AI output into Slack and email components, derives an email subject from the generated HTML, and sends both outputs in parallel.

### Nodes Involved
- Parse AI Output
- Send to Slack
- Send Email

### Node Details

#### Parse AI Output
- **Type and technical role:** `n8n-nodes-base.code`  
  Post-processes the agent’s text output into structured fields usable by delivery nodes.
- **Configuration choices:**
  - Reads the first input item’s `json.output`
  - Throws an error if no output is present
  - Splits the content using `---SEPARATOR---`
  - Throws an error if the delimiter is missing
  - Extracts the email subject from the first `<h2>` in the HTML section
  - Falls back to `Paid Ads Weekly Report` if no `<h2>` is found
  - Returns:
    - `slackMessage`
    - `emailSubject`
    - `emailHtml`
- **Key expressions or variables used:**
  - `const aiOutput = $input.first().json.output;`
  - delimiter: `---SEPARATOR---`
- **Input and output connections:**
  - Input from **Paid Ads Reporting Agent**
  - Output to both **Send to Slack** and **Send Email**
- **Version-specific requirements:**
  - Type version `2`
  - Requires Code node JavaScript runtime support.
- **Edge cases or potential failure types:**
  - Missing `output` property if the agent changes response structure.
  - Missing delimiter if the model fails formatting.
  - Regex subject extraction may fail if HTML heading structure changes.
  - If HTML includes unusual markup, subject extraction may become noisy.
- **Sub-workflow reference:** None.

#### Send to Slack
- **Type and technical role:** `n8n-nodes-base.slack`  
  Sends the Slack summary message to a specified Slack channel.
- **Configuration choices:**
  - Authentication: OAuth2
  - Message text:
    - `{{ $json.slackMessage }}`
  - Destination selection mode: channel
  - Channel ID is hard-coded as `C0AELRTHZAA`
- **Key expressions or variables used:**
  - `{{ $json.slackMessage }}`
- **Input and output connections:**
  - Input from **Parse AI Output**
  - No downstream connection
- **Version-specific requirements:**
  - Type version `2.2`
  - Requires a Slack app/credential with permission to post to the selected channel.
- **Edge cases or potential failure types:**
  - Invalid or unauthorized channel ID.
  - Bot not invited to channel.
  - OAuth scope issues.
  - Slack formatting may render differently than expected if HTML-like characters appear in text.
- **Sub-workflow reference:** None.

#### Send Email
- **Type and technical role:** `n8n-nodes-base.gmail`  
  Sends the HTML report using Gmail.
- **Configuration choices:**
  - Recipient is statically set to `user@example.com`
  - Subject:
    - `{{ $json.emailSubject }}`
  - Message body:
    - `{{ $json.emailHtml }}`
  - No additional send options configured
- **Key expressions or variables used:**
  - `{{ $json.emailSubject }}`
  - `{{ $json.emailHtml }}`
- **Input and output connections:**
  - Input from **Parse AI Output**
  - No downstream connection
- **Version-specific requirements:**
  - Type version `2.2`
  - Requires Gmail OAuth2 credentials and a compatible n8n Gmail node setup.
- **Edge cases or potential failure types:**
  - Gmail OAuth authorization failure.
  - Sending limits, domain restrictions, or blocked account access.
  - Some email clients may sanitize or restyle HTML.
  - If `emailHtml` is malformed, email rendering can degrade.
- **Sub-workflow reference:** None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Every Monday 9 AM | Schedule Trigger | Weekly workflow trigger at Monday 9 AM |  | Get Current Date | ## 1️⃣ Scheduled Execution<br>### What this section does<br>Triggers the workflow every **Monday at 9 AM** and captures the current date so the AI Agent can calculate the correct 7-day reporting windows (current week vs. previous week).<br>### Change the schedule (optional)<br>- Click the "Every Monday 9 AM" node<br>- Click on the Cron Expression field<br>- Modify the schedule (e.g., change to daily, weekly on different day, or custom time) |
| Get Current Date | Date & Time | Produces current date context for the AI prompt | Every Monday 9 AM | Paid Ads Reporting Agent | ## 1️⃣ Scheduled Execution<br>### What this section does<br>Triggers the workflow every **Monday at 9 AM** and captures the current date so the AI Agent can calculate the correct 7-day reporting windows (current week vs. previous week).<br>### Change the schedule (optional)<br>- Click the "Every Monday 9 AM" node<br>- Click on the Cron Expression field<br>- Modify the schedule (e.g., change to daily, weekly on different day, or custom time) |
| Paid Ads Reporting Agent | LangChain Agent | Discovers platforms, retrieves metrics via Databox MCP, computes WoW comparisons, and formats report text/HTML | Get Current Date; OpenAI Chat Model; Databox MCP Tool | Parse AI Output | ## 2️⃣ AI Agent + Databox MCP Setup<br>### What this section/Agent does<br>The AI Agent connects to Databox via MCP and intelligently auto-discovers your connected paid ads platforms, fetches data across all of them for 6 key metrics, aggregates performance, calculates week-over-week changes, and formats professional reports for Slack and Email.<br>### What you need to do ⚠️<br>1. Click the **OpenAI Chat Model** node → add your `API key` credential<br>   - You can also replace this node with the **OpenAI Chat Model** node<br>2. Click the **Databox MCP Tool** node → set Authentication to `OAuth2` → authorize with your Databox account<br>3. **Ensure at least one paid ads platform is connected** in your Databox account<br>### How to connect Databox MCP Tool in n8n<br>@[youtube](892KtXhv-vI) |
| OpenAI Chat Model | OpenAI Chat Model | Supplies the LLM used by the agent |  | Paid Ads Reporting Agent | ## 2️⃣ AI Agent + Databox MCP Setup<br>### What this section/Agent does<br>The AI Agent connects to Databox via MCP and intelligently auto-discovers your connected paid ads platforms, fetches data across all of them for 6 key metrics, aggregates performance, calculates week-over-week changes, and formats professional reports for Slack and Email.<br>### What you need to do ⚠️<br>1. Click the **OpenAI Chat Model** node → add your `API key` credential<br>   - You can also replace this node with the **OpenAI Chat Model** node<br>2. Click the **Databox MCP Tool** node → set Authentication to `OAuth2` → authorize with your Databox account<br>3. **Ensure at least one paid ads platform is connected** in your Databox account<br>### How to connect Databox MCP Tool in n8n<br>@[youtube](892KtXhv-vI) |
| Databox MCP Tool | MCP Client Tool | Exposes Databox MCP functions to the AI Agent |  | Paid Ads Reporting Agent | ## 2️⃣ AI Agent + Databox MCP Setup<br>### What this section/Agent does<br>The AI Agent connects to Databox via MCP and intelligently auto-discovers your connected paid ads platforms, fetches data across all of them for 6 key metrics, aggregates performance, calculates week-over-week changes, and formats professional reports for Slack and Email.<br>### What you need to do ⚠️<br>1. Click the **OpenAI Chat Model** node → add your `API key` credential<br>   - You can also replace this node with the **OpenAI Chat Model** node<br>2. Click the **Databox MCP Tool** node → set Authentication to `OAuth2` → authorize with your Databox account<br>3. **Ensure at least one paid ads platform is connected** in your Databox account<br>### How to connect Databox MCP Tool in n8n<br>@[youtube](892KtXhv-vI) |
| Parse AI Output | Code | Splits AI output into Slack text and email HTML/subject | Paid Ads Reporting Agent | Send to Slack; Send Email | ## 3️⃣ Output & Notification<br>### What this section does<br>The Parse AI Output node splits the Agent's response into two formatted reports - a concise Slack summary with top-level metrics and AI-generated insights, and a rich HTML email with individual platform tables and color-coded week-over-week changes - then delivers both simultaneously.<br>### What you need to do ⚠️<br>- **Slack**: Connect your account in **Send to Slack** node and set the Slack `channel` for report delivery<br>- **Email**: Add your Gmail/SMTP credentials to the **Send Email** node<br>- **Optional**: Add a Microsoft Teams, Discord, or Telegram node for additional outputs |
| Send to Slack | Slack | Posts the summary report to Slack | Parse AI Output |  | ## 3️⃣ Output & Notification<br>### What this section does<br>The Parse AI Output node splits the Agent's response into two formatted reports - a concise Slack summary with top-level metrics and AI-generated insights, and a rich HTML email with individual platform tables and color-coded week-over-week changes - then delivers both simultaneously.<br>### What you need to do ⚠️<br>- **Slack**: Connect your account in **Send to Slack** node and set the Slack `channel` for report delivery<br>- **Email**: Add your Gmail/SMTP credentials to the **Send Email** node<br>- **Optional**: Add a Microsoft Teams, Discord, or Telegram node for additional outputs |
| Send Email | Gmail | Sends the detailed HTML report by email | Parse AI Output |  | ## 3️⃣ Output & Notification<br>### What this section does<br>The Parse AI Output node splits the Agent's response into two formatted reports - a concise Slack summary with top-level metrics and AI-generated insights, and a rich HTML email with individual platform tables and color-coded week-over-week changes - then delivers both simultaneously.<br>### What you need to do ⚠️<br>- **Slack**: Connect your account in **Send to Slack** node and set the Slack `channel` for report delivery<br>- **Email**: Add your Gmail/SMTP credentials to the **Send Email** node<br>- **Optional**: Add a Microsoft Teams, Discord, or Telegram node for additional outputs |
| Sticky Note | Sticky Note | Visual documentation for overall workflow purpose and requirements |  |  | ## Automated Paid Ads Weekly Performance Report via Databox MCP<br>Automate your paid advertising performance reporting across all platforms with AI-powered insights delivered every Monday morning. This workflow connects to your Databox account via MCP, automatically discovers which paid ads platforms you have connected, fetches 6 key metrics from each platform, calculates week-over-week performance changes, and sends beautifully formatted consolidated reports to both Slack and email-completely hands-free.<br>### What you'll get<br>- 📊 Automated weekly reports<br>- 📈 6 key metrics tracked per platform: Cost/Spend, Clicks, CPC, CTR, Impressions, Conversions<br>- 🎯 Week-over-week % changes<br>- 💬 Summary with highlights and AI-generated insights<br>- 📧 HTML email report<br>- ⏰ Scheduled automation<br>### How it works<br>`Schedule Trigger` → `AI Agent (auto-discovers connected platforms via Databox MCP)` → `Aggregates metrics across all platforms` → `Send report to Slack + Email`<br>### What you need<br>- Databox account with at least one paid ads platform connected → Free plan available: https://databox.com/?ref=n8n<br>- Claude or ChatGPT API key<br>- Slack account (optional)<br>- Gmail account (optional)<br>### Supported Platforms<br>Facebook Ads • Google Ads • LinkedIn Ads • YouTube Ads • Reddit Ads • TikTok Ads • Snapchat Ads • Microsoft Advertising • X (Twitter) Ads • Pinterest Ads |
| Sticky Note 1 | Sticky Note | Visual documentation for schedule section |  |  |  |
| Sticky Note 2 | Sticky Note | Visual documentation for AI/Databox setup section |  |  |  |
| Sticky Note 3 | Sticky Note | Visual documentation for output section |  |  |  |
| Sticky Note 4 | Sticky Note | Visual documentation with video link |  |  | ## How it works<br>@[youtube](Y3bJsPWtSIA) |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it something like: `Automated paid ads weekly performance report`.

2. **Add a Schedule Trigger node**
   - Node type: **Schedule Trigger**
   - Rename it to: `Every Monday 9 AM`
   - Configure a cron-based schedule:
     - Cron expression: `0 9 * * 1`
   - This runs every Monday at 9:00 AM.
   - If needed, verify your n8n instance timezone so Monday 9 AM matches your local expectation.

3. **Add a Date & Time node**
   - Node type: **Date & Time**
   - Rename it to: `Get Current Date`
   - Leave default options unless your environment requires a specific format.
   - Connect:
     - `Every Monday 9 AM` → `Get Current Date`

4. **Add an OpenAI Chat Model node**
   - Node type: **OpenAI Chat Model**
   - Rename it to: `OpenAI Chat Model`
   - Select model:
     - `gpt-5-mini`
   - Add your **OpenAI API** credentials.
   - Leave advanced options empty unless you need stricter model tuning.

5. **Add a Databox MCP Tool node**
   - Node type: **MCP Client Tool**
   - Rename it to: `Databox MCP Tool`
   - Configure:
     - **Endpoint URL:** `https://mcp.databox.com/mcp`
     - **Authentication:** OAuth2 / MCP OAuth2 API
   - Create or select Databox OAuth2 credentials.
   - Authorize with your Databox account.
   - Make sure at least one paid ads source is already connected in Databox.

6. **Add an AI Agent node**
   - Node type: **AI Agent**
   - Rename it to: `Paid Ads Reporting Agent`
   - Set prompt mode to a defined prompt.
   - In the main prompt, paste this logic in equivalent form:
     - Include today’s date using:
       - `Today's Date: {{ $json.formattedDate }}`
     - Ask the agent to generate a weekly report for all connected paid advertising platforms.
     - Instruct it to determine:
       - last 7 days,
       - previous 7 days.
     - Ask it to retrieve, for each connected platform:
       - Cost/Spend
       - Clicks
       - CPC
       - CTR
       - Impressions
       - Conversions
     - Ask it to calculate:
       - WoW % per metric and platform
       - total spend, clicks, impressions, conversions
       - average CPC
       - average CTR
     - Require output in two sections separated by:
       - `---SEPARATOR---`
     - First section = Slack message
     - Second section = HTML email
   - In the **system message**, add the operating instructions reflected by the original workflow:
     - call account and data source discovery tools,
     - identify supported paid ads platforms,
     - fetch metrics over a 14-day window with weekly granularity,
     - skip disconnected/no-data platforms silently,
     - produce a no-activity report if needed,
     - use `N/A` when previous week is zero,
     - use exact formatting rules for Slack and HTML.
   - Enable:
     - **Always Output Data**
   - Disable:
     - **Retry On Fail**
   - Connect:
     - `Get Current Date` → `Paid Ads Reporting Agent`
     - `OpenAI Chat Model` → AI language model input on `Paid Ads Reporting Agent`
     - `Databox MCP Tool` → AI tool input on `Paid Ads Reporting Agent`

7. **Add a Code node**
   - Node type: **Code**
   - Rename it to: `Parse AI Output`
   - Set JavaScript code to equivalent logic:
     - read the agent output from the incoming item,
     - throw an error if empty,
     - split by `---SEPARATOR---`,
     - throw an error if the separator is missing,
     - extract the `<h2>` title from the HTML as email subject,
     - default subject to `Paid Ads Weekly Report`,
     - return:
       - `slackMessage`
       - `emailSubject`
       - `emailHtml`
   - Use this logic structure:
     - input: agent text in `json.output`
     - output: one JSON item with parsed fields
   - Connect:
     - `Paid Ads Reporting Agent` → `Parse AI Output`

8. **Add a Slack node**
   - Node type: **Slack**
   - Rename it to: `Send to Slack`
   - Configure authentication:
     - OAuth2
   - Connect your Slack credentials.
   - Configure message destination:
     - Select a channel
     - Set the desired channel ID
   - Set the message text expression to:
     - `{{ $json.slackMessage }}`
   - Connect:
     - `Parse AI Output` → `Send to Slack`

9. **Add a Gmail node**
   - Node type: **Gmail**
   - Rename it to: `Send Email`
   - Connect your Gmail OAuth2 credentials.
   - Set:
     - **To:** your recipient email address
     - **Subject:** `{{ $json.emailSubject }}`
     - **Message/HTML body:** `{{ $json.emailHtml }}`
   - Connect:
     - `Parse AI Output` → `Send Email`

10. **Verify parallel outputs**
   - Ensure `Parse AI Output` has two outgoing main connections:
     - one to Slack,
     - one to Gmail.
   - This allows both notifications to send from the same parsed item.

11. **Optionally add documentation sticky notes**
   - Add sticky notes for:
     - overall workflow summary,
     - scheduling instructions,
     - AI/Databox setup guidance,
     - output delivery notes.
   - These do not affect execution.

12. **Credential checklist**
   - **Databox MCP OAuth2**
     - Must authorize successfully against `https://mcp.databox.com/mcp`
   - **OpenAI**
     - Valid API key with access to the chosen model
   - **Slack OAuth2**
     - Bot/app must be allowed to post into the selected channel
   - **Gmail OAuth2**
     - Must have permission to send mail from the chosen Google account

13. **Testing recommendations**
   - Execute manually once before activation.
   - Confirm the agent output includes `---SEPARATOR---`.
   - Check Slack formatting.
   - Check HTML email rendering in your mail client.
   - Test behavior when:
     - no platforms are connected,
     - one platform has no data,
     - previous week values are zero.

14. **Activate the workflow**
   - Turn the workflow on only after:
     - all credentials are valid,
     - channel and recipient values are updated,
     - Databox paid sources are connected.

### Sub-workflow setup
This workflow does **not** use any Execute Workflow or sub-workflow node. No external n8n workflow dependency is required.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Databox account with at least one paid ads platform connected is required; free plan available | https://databox.com/?ref=n8n |
| Supported platforms mentioned in the workflow note: Facebook Ads, Google Ads, LinkedIn Ads, YouTube Ads, Reddit Ads, TikTok Ads, Snapchat Ads, Microsoft Advertising, X (Twitter) Ads, Pinterest Ads | Overall workflow scope |
| Video reference for connecting Databox MCP Tool in n8n | `@[youtube](892KtXhv-vI)` |
| Video reference: “How it works” | `@[youtube](Y3bJsPWtSIA)` |
| Slack and Gmail are described as optional in the note, but the current workflow includes both delivery nodes and will try to execute both unless you remove or disable one branch | Deployment consideration |
| The workflow title in JSON is `Automated paid ads weekly performance report`, while the supplied title/description labels it as sending weekly Databox paid ads performance reports to Slack and email | Naming/context note |
| The agent prompt says “Claude or ChatGPT API key” in notes, but the actual configured model node is OpenAI | Configuration clarification |
| The prompt depends heavily on the AI strictly following formatting rules; the `Parse AI Output` code will fail if the separator is missing | Reliability consideration |