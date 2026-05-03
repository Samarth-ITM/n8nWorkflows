Send a daily paid acquisition and website intelligence report with Databox, GPT-4o and Gmail

https://n8nworkflows.xyz/workflows/send-a-daily-paid-acquisition-and-website-intelligence-report-with-databox--gpt-4o-and-gmail-14323


# Send a daily paid acquisition and website intelligence report with Databox, GPT-4o and Gmail

# 1. Workflow Overview

This workflow generates and emails a daily paid acquisition intelligence report by combining Databox data retrieval, GPT-4o-based analysis, and Gmail delivery.

Its purpose is to help marketers, growth teams, and agency operators receive a recurring morning briefing that explains:

- how website performance changed over the last 7 days versus the previous 7 days,
- how paid acquisition channels performed over the same comparison window,
- what cross-channel relationships exist between paid traffic and website outcomes,
- and what actions should be taken next.

The workflow is organized into three main logical blocks.

## 1.1 Scheduled Trigger and Date Context

The workflow starts on a fixed daily schedule at 8:00 AM. It then generates a date value so all downstream AI agents reference the same “today” context for period calculation.

## 1.2 Sequential AI Analysis with Databox MCP

Three AI agents run in sequence:

1. **Website Analysis Agent**  
   Uses the Databox MCP tool to inspect website analytics sources and summarize website behavior.

2. **Paid Acquisition Agent**  
   Uses the Databox MCP tool to inspect connected paid media sources and summarize platform-level paid performance.

3. **Correlation Agent**  
   Receives the outputs of the previous two agents and produces the final HTML report. This third agent does not call Databox directly.

This sequence matters because each downstream agent depends on earlier outputs:
- Agent 2 receives Agent 1’s result as context.
- Agent 3 receives both Agent 1 and Agent 2 outputs.

## 1.3 Email Formatting and Delivery

After the HTML report is created, a Code node wraps it into an email subject/body payload. A Gmail node then sends the report to the configured recipient.

---

# 2. Block-by-Block Analysis

## 2.1 Block 1 - Schedule and Shared Date Context

### Overview

This block triggers the workflow every day at 8:00 AM and establishes a single date context used by all analysis steps. It ensures the reporting period is interpreted consistently across all agents.

### Nodes Involved

- **Every Day 8 AM**
- **Get Current Date**

### Node Details

#### 2.1.1 Every Day 8 AM

- **Type and technical role:**  
  `n8n-nodes-base.scheduleTrigger`  
  Entry-point trigger node that launches the workflow on a cron schedule.

- **Configuration choices:**  
  Configured with cron expression `0 8 * * *`, which means every day at 8:00 AM.

- **Key expressions or variables used:**  
  None.

- **Input and output connections:**  
  - Input: none, as this is a trigger
  - Output: to **Get Current Date**

- **Version-specific requirements:**  
  Uses node type version `1.2`.

- **Edge cases or potential failure types:**  
  - Workflow will not run unless activated in n8n.
  - Timezone behavior depends on the n8n instance/workflow timezone settings.
  - If users expect local time and the server is running in another timezone, the report may arrive at the wrong hour.

- **Sub-workflow reference:**  
  None.

---

#### 2.1.2 Get Current Date

- **Type and technical role:**  
  `n8n-nodes-base.dateTime`  
  Generates a formatted current date value for use in downstream prompts.

- **Configuration choices:**  
  The node uses default options. In practice, this means it produces date-related fields such as `formattedDate` that are referenced later in prompts.

- **Key expressions or variables used:**  
  Downstream nodes use:
  - `{{ $json.formattedDate }}`
  - `{{ $('Get Current Date').item.json.formattedDate }}`

- **Input and output connections:**  
  - Input: from **Every Day 8 AM**
  - Output: to **Website Analysis Agent**

- **Version-specific requirements:**  
  Uses node type version `2`.

- **Edge cases or potential failure types:**  
  - If the Date & Time node output format changes across n8n versions or due to manual reconfiguration, downstream expressions referencing `formattedDate` may fail or return empty values.
  - If timezone handling matters for reporting windows, confirm the node and workflow timezone align with the business reporting timezone.

- **Sub-workflow reference:**  
  None.

---

## 2.2 Block 2 - Website Analytics Analysis via Databox MCP

### Overview

This block asks an AI agent to inspect connected website analytics sources in Databox, retrieve key traffic and engagement metrics for the last 7 days versus the previous 7 days, and summarize the results in plain text.

### Nodes Involved

- **Website Analysis Agent**
- **OpenAI Chat Model 1**
- **Databox MCP Tool**

### Node Details

#### 2.2.1 Website Analysis Agent

- **Type and technical role:**  
  `@n8n/n8n-nodes-langchain.agent`  
  AI agent node that combines prompt instructions, a chat model, and an MCP tool to retrieve and analyze website analytics data.

- **Configuration choices:**  
  The prompt includes:
  - today’s formatted date from **Get Current Date**
  - a task asking for website performance over the last 7 days compared with the previous 7 days
  - focus areas: traffic volume, engagement quality, and conversion metrics

  The system message is detailed and instructs the agent to:
  1. call `list_accounts`,
  2. call `list_data_sources`,
  3. identify website analytics sources,
  4. fetch metrics for both periods,
  5. calculate week-over-week changes,
  6. return structured plain text only.

  It is explicitly told not to output HTML.

- **Key expressions or variables used:**  
  - `{{ $json.formattedDate }}`

- **Input and output connections:**  
  - Main input: from **Get Current Date**
  - AI language model input: from **OpenAI Chat Model 1**
  - AI tool input: from **Databox MCP Tool**
  - Main output: to **Paid Acquisition Agent**

- **Version-specific requirements:**  
  Uses node type version `3`. Requires compatible LangChain/AI features available in the n8n version used.

- **Edge cases or potential failure types:**  
  - OpenAI authentication failure
  - Databox MCP OAuth2 authentication failure
  - MCP endpoint unavailability or timeout
  - Tool discovery failure if Databox account has no accessible sources
  - Model may return less structured output than expected
  - Metrics named differently across analytics sources may require agent interpretation
  - If no website source is connected, expected fallback output is: `"No website analytics data available."`

- **Sub-workflow reference:**  
  None.

---

#### 2.2.2 OpenAI Chat Model 1

- **Type and technical role:**  
  `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
  Supplies the GPT-4o model used by the Website Analysis Agent.

- **Configuration choices:**  
  - Model: `gpt-4o`
  - Max tokens: `2048`
  - No built-in tools configured in the model node itself

- **Key expressions or variables used:**  
  None.

- **Input and output connections:**  
  - Output via `ai_languageModel` to **Website Analysis Agent**

- **Version-specific requirements:**  
  Uses node type version `1.3`. Requires valid OpenAI credentials.

- **Edge cases or potential failure types:**  
  - Invalid API key
  - Quota exhaustion
  - Model availability changes
  - Token limit may truncate output if the result is very verbose across many data sources

- **Sub-workflow reference:**  
  None.

---

#### 2.2.3 Databox MCP Tool

- **Type and technical role:**  
  `@n8n/n8n-nodes-langchain.mcpClientTool`  
  Exposes Databox MCP capabilities to the AI agent so it can query connected accounts and data sources.

- **Configuration choices:**  
  - Endpoint URL: `https://mcp.databox.com/mcp`
  - Authentication: `mcpOAuth2Api`

- **Key expressions or variables used:**  
  None.

- **Input and output connections:**  
  - Output via `ai_tool` to **Website Analysis Agent**

- **Version-specific requirements:**  
  Uses node type version `1.2`. Requires MCP/OAuth2-compatible n8n support and a valid Databox OAuth2 credential.

- **Edge cases or potential failure types:**  
  - OAuth authorization not completed
  - Revoked Databox access token
  - MCP server/network issues
  - Databox account may not expose expected sources or metrics
  - API/tool schema changes may affect agent behavior

- **Sub-workflow reference:**  
  None.

---

## 2.3 Block 3 - Paid Acquisition Analysis via Databox MCP

### Overview

This block asks a second AI agent to inspect paid media sources in Databox, retrieve platform-level paid metrics for the same comparison window, and generate a structured plain-text summary of paid acquisition performance.

### Nodes Involved

- **Paid Acquisition Agent**
- **OpenAI Chat Model 2**
- **Databox MCP Tool 2**

### Node Details

#### 2.3.1 Paid Acquisition Agent

- **Type and technical role:**  
  `@n8n/n8n-nodes-langchain.agent`  
  AI agent that analyzes paid channel performance by combining prompt instructions, GPT-4o, and the Databox MCP tool.

- **Configuration choices:**  
  Its prompt includes:
  - today’s date from **Get Current Date**
  - the website analysis output from Agent 1
  - instructions to fetch paid acquisition data for the same periods

  The system message instructs the agent to:
  1. call `list_accounts`,
  2. call `list_data_sources`,
  3. identify connected paid advertising platforms,
  4. fetch spend, clicks, CPC, CTR, impressions, and ROAS/conversions,
  5. calculate week-over-week change,
  6. compute aggregated totals across all paid platforms,
  7. return plain text only.

- **Key expressions or variables used:**  
  - `{{ $('Get Current Date').item.json.formattedDate }}`
  - `{{ $json.output }}`  
    Here, `$json.output` refers to the output of **Website Analysis Agent** because this node receives that node’s main output.

- **Input and output connections:**  
  - Main input: from **Website Analysis Agent**
  - AI language model input: from **OpenAI Chat Model 2**
  - AI tool input: from **Databox MCP Tool 2**
  - Main output: to **Correlation Agent**

- **Version-specific requirements:**  
  Uses node type version `3`.

- **Edge cases or potential failure types:**  
  - Same AI/API issues as Agent 1
  - Databox may contain multiple ad accounts/platforms with uneven metric availability
  - Some platforms may expose conversions but not ROAS, or vice versa
  - Large numbers of connected platforms could increase prompt/tool complexity
  - If no paid platforms are connected, expected fallback output is: `"No paid advertising data available."`

- **Sub-workflow reference:**  
  None.

---

#### 2.3.2 OpenAI Chat Model 2

- **Type and technical role:**  
  `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
  GPT-4o model used by the Paid Acquisition Agent.

- **Configuration choices:**  
  - Model: `gpt-4o`
  - Max tokens: `2048`

- **Key expressions or variables used:**  
  None.

- **Input and output connections:**  
  - Output via `ai_languageModel` to **Paid Acquisition Agent**

- **Version-specific requirements:**  
  Uses node type version `1.3`.

- **Edge cases or potential failure types:**  
  - OpenAI credential failure
  - Token limit pressure if many paid platforms are connected
  - Model output formatting inconsistency

- **Sub-workflow reference:**  
  None.

---

#### 2.3.3 Databox MCP Tool 2

- **Type and technical role:**  
  `@n8n/n8n-nodes-langchain.mcpClientTool`  
  A second Databox MCP tool node dedicated to the Paid Acquisition Agent.

- **Configuration choices:**  
  - Endpoint URL: `https://mcp.databox.com/mcp`
  - Authentication: `mcpOAuth2Api`

- **Key expressions or variables used:**  
  None.

- **Input and output connections:**  
  - Output via `ai_tool` to **Paid Acquisition Agent**

- **Version-specific requirements:**  
  Uses node type version `1.2`.

- **Edge cases or potential failure types:**  
  Same as the first Databox MCP tool:
  - auth failures
  - endpoint issues
  - data source availability mismatch
  - schema/tool changes

- **Sub-workflow reference:**  
  None.

---

## 2.4 Block 4 - Cross-Channel Correlation and HTML Report Generation

### Overview

This block receives the plain-text outputs from the website and paid acquisition agents, then asks a third AI agent to synthesize them into a branded HTML email report with correlation analysis and recommendations.

### Nodes Involved

- **Correlation Agent**
- **OpenAI Chat Model 3**

### Node Details

#### 2.4.1 Correlation Agent

- **Type and technical role:**  
  `@n8n/n8n-nodes-langchain.agent`  
  AI agent that performs synthesis and report generation. Unlike the earlier agents, this node does not use external tools.

- **Configuration choices:**  
  The prompt injects:
  - today’s date from **Get Current Date**
  - website analysis from **Website Analysis Agent**
  - paid acquisition analysis from **Paid Acquisition Agent**

  It explicitly asks for:
  1. executive summary,
  2. website performance table,
  3. paid acquisition per-platform tables,
  4. correlation analysis,
  5. exactly 3 actionable recommendations,
  6. footer.

  It must return only the HTML body content starting with a `<div>` tag.

  The system message imposes formatting rules:
  - Databox blue `#3164FA` for headings/table headers
  - green/red colors for WoW changes
  - Helvetica Neue / Arial
  - box styling for key sections
  - no `<html>`, `<head>`, or `<body>`
  - numbers and currency formatting requirements
  - graceful handling of missing data

- **Key expressions or variables used:**  
  - `{{ $('Get Current Date').item.json.formattedDate }}`
  - `{{ $('Website Analysis Agent').item.json.output }}`
  - `{{ $json.output }}`  
    In this node, `$json.output` refers to the main output of **Paid Acquisition Agent**.

- **Input and output connections:**  
  - Main input: from **Paid Acquisition Agent**
  - AI language model input: from **OpenAI Chat Model 3**
  - Main output: to **Prepare Email**

- **Version-specific requirements:**  
  Uses node type version `3`.

- **Edge cases or potential failure types:**  
  - If upstream agent outputs are empty or malformed, the HTML report may be weak or incomplete
  - The model may still produce non-HTML text despite the instruction
  - It may produce partial HTML or omit the required `<div>` start
  - Correlation quality depends entirely on the fidelity of the two upstream summaries
  - If website or paid data is missing, the prompt instructs graceful handling, but the output may still vary in completeness

- **Sub-workflow reference:**  
  None.

---

#### 2.4.2 OpenAI Chat Model 3

- **Type and technical role:**  
  `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
  GPT-4o model used for final HTML synthesis.

- **Configuration choices:**  
  - Model: `gpt-4o`
  - Max tokens: `4096`  
    The higher limit is sensible because this node generates a full HTML report.

- **Key expressions or variables used:**  
  None.

- **Input and output connections:**  
  - Output via `ai_languageModel` to **Correlation Agent**

- **Version-specific requirements:**  
  Uses node type version `1.3`.

- **Edge cases or potential failure types:**  
  - OpenAI auth/quota/model availability issues
  - Long generated HTML could still approach token constraints if upstream summaries are very large
  - Model may emit formatting that works poorly in email clients

- **Sub-workflow reference:**  
  None.

---

## 2.5 Block 5 - Email Payload Preparation and Gmail Delivery

### Overview

This block takes the HTML report, generates a dated subject line, and sends it via Gmail to the configured recipient.

### Nodes Involved

- **Prepare Email**
- **Send Email**

### Node Details

#### 2.5.1 Prepare Email

- **Type and technical role:**  
  `n8n-nodes-base.code`  
  JavaScript transformation node that packages the HTML report into a clean email payload.

- **Configuration choices:**  
  The code:
  - reads `output` from the incoming item,
  - creates a human-readable date string using JavaScript `toLocaleDateString('en-US', ...)`,
  - constructs:
    - `subject`: `Daily Paid Acquisition Report - <Month Day, Year>`
    - `htmlBody`: the agent output

- **Key expressions or variables used:**  
  In code:
  - `$input.first().json.output`

- **Input and output connections:**  
  - Input: from **Correlation Agent**
  - Output: to **Send Email**

- **Version-specific requirements:**  
  Uses node type version `2`.

- **Edge cases or potential failure types:**  
  - If `output` is empty, email body will be blank
  - Date is generated using runtime server locale behavior and current execution time, not the earlier Date & Time node output
  - If consistency with the exact reporting timezone is critical, using a separate JS `new Date()` may produce slight mismatches around midnight or timezone boundaries

- **Sub-workflow reference:**  
  None.

---

#### 2.5.2 Send Email

- **Type and technical role:**  
  `n8n-nodes-base.gmail`  
  Sends the final email through Gmail OAuth2.

- **Configuration choices:**  
  - Recipient: `user@example.com` placeholder
  - Subject: `{{ $json.subject }}`
  - Message/body: `{{ $json.htmlBody }}`
  - Uses Gmail OAuth2 credentials

- **Key expressions or variables used:**  
  - `{{ $json.subject }}`
  - `{{ $json.htmlBody }}`

- **Input and output connections:**  
  - Input: from **Prepare Email**
  - Output: none

- **Version-specific requirements:**  
  Uses node type version `2.2`. Requires a valid Gmail OAuth2 credential in n8n.

- **Edge cases or potential failure types:**  
  - Gmail OAuth token expired or revoked
  - Recipient address not updated from placeholder
  - Gmail sending limits or account restrictions
  - HTML rendering may vary by email client
  - If the body is very long or malformed HTML, the final email may render poorly

- **Sub-workflow reference:**  
  None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Every Day 8 AM | n8n-nodes-base.scheduleTrigger | Daily cron entry point |  | Get Current Date | ## 1 - Schedule<br>### What this section does<br>Triggers the workflow every day at 8 AM and captures today's date so all three agents use consistent reporting windows.<br>### Change the schedule (optional)<br>- Click the "Every Monday 8 AM" node<br>- Click on the Cron Expression field<br>- Modify the schedule (e.g., change to daily, weekly on different day, or custom time) |
| Get Current Date | n8n-nodes-base.dateTime | Generates current date context for prompts | Every Day 8 AM | Website Analysis Agent | ## 1 - Schedule<br>### What this section does<br>Triggers the workflow every day at 8 AM and captures today's date so all three agents use consistent reporting windows.<br>### Change the schedule (optional)<br>- Click the "Every Monday 8 AM" node<br>- Click on the Cron Expression field<br>- Modify the schedule (e.g., change to daily, weekly on different day, or custom time) |
| Website Analysis Agent | @n8n/n8n-nodes-langchain.agent | Retrieves and summarizes website analytics data | Get Current Date; OpenAI Chat Model 1; Databox MCP Tool | Paid Acquisition Agent | ## 2 - AI Agents + Databox MCP Setup<br>### What this section does<br>Three agents run in sequence:<br>- **Agent 1 - Website Analyst**: fetches sessions, bounce rate, pages per session, goal completions from your website analytics source<br>- **Agent 2 - Paid Acquisition Analyst**: fetches spend, CPC, CTR, ROAS and impressions for every connected paid platform<br>- **Agent 3 - Correlation Analyst**: receives both outputs, finds cross-channel patterns, ranks channel efficiency, and writes the final HTML report (no MCP calls needed)<br>### What you need to do<br>1. Click each **OpenAI Chat Model** node and add your API key<br>- You can replace with an **Anthropic Chat Model** node<br>2. Click each **Databox MCP Tool** node - set Authentication to `OAuth2` - authorize with your Databox account<br>3. Ensure at least one paid ads platform and a website analytics source are connected in Databox |
| Databox MCP Tool | @n8n/n8n-nodes-langchain.mcpClientTool | Provides Databox MCP tool access to the website agent |  | Website Analysis Agent | ## 2 - AI Agents + Databox MCP Setup<br>### What this section does<br>Three agents run in sequence:<br>- **Agent 1 - Website Analyst**: fetches sessions, bounce rate, pages per session, goal completions from your website analytics source<br>- **Agent 2 - Paid Acquisition Analyst**: fetches spend, CPC, CTR, ROAS and impressions for every connected paid platform<br>- **Agent 3 - Correlation Analyst**: receives both outputs, finds cross-channel patterns, ranks channel efficiency, and writes the final HTML report (no MCP calls needed)<br>### What you need to do<br>1. Click each **OpenAI Chat Model** node and add your API key<br>- You can replace with an **Anthropic Chat Model** node<br>2. Click each **Databox MCP Tool** node - set Authentication to `OAuth2` - authorize with your Databox account<br>3. Ensure at least one paid ads platform and a website analytics source are connected in Databox |
| Paid Acquisition Agent | @n8n/n8n-nodes-langchain.agent | Retrieves and summarizes paid ads data | Website Analysis Agent; OpenAI Chat Model 2; Databox MCP Tool 2 | Correlation Agent | ## 2 - AI Agents + Databox MCP Setup<br>### What this section does<br>Three agents run in sequence:<br>- **Agent 1 - Website Analyst**: fetches sessions, bounce rate, pages per session, goal completions from your website analytics source<br>- **Agent 2 - Paid Acquisition Analyst**: fetches spend, CPC, CTR, ROAS and impressions for every connected paid platform<br>- **Agent 3 - Correlation Analyst**: receives both outputs, finds cross-channel patterns, ranks channel efficiency, and writes the final HTML report (no MCP calls needed)<br>### What you need to do<br>1. Click each **OpenAI Chat Model** node and add your API key<br>- You can replace with an **Anthropic Chat Model** node<br>2. Click each **Databox MCP Tool** node - set Authentication to `OAuth2` - authorize with your Databox account<br>3. Ensure at least one paid ads platform and a website analytics source are connected in Databox |
| Databox MCP Tool 2 | @n8n/n8n-nodes-langchain.mcpClientTool | Provides Databox MCP tool access to the paid acquisition agent |  | Paid Acquisition Agent | ## 2 - AI Agents + Databox MCP Setup<br>### What this section does<br>Three agents run in sequence:<br>- **Agent 1 - Website Analyst**: fetches sessions, bounce rate, pages per session, goal completions from your website analytics source<br>- **Agent 2 - Paid Acquisition Analyst**: fetches spend, CPC, CTR, ROAS and impressions for every connected paid platform<br>- **Agent 3 - Correlation Analyst**: receives both outputs, finds cross-channel patterns, ranks channel efficiency, and writes the final HTML report (no MCP calls needed)<br>### What you need to do<br>1. Click each **OpenAI Chat Model** node and add your API key<br>- You can replace with an **Anthropic Chat Model** node<br>2. Click each **Databox MCP Tool** node - set Authentication to `OAuth2` - authorize with your Databox account<br>3. Ensure at least one paid ads platform and a website analytics source are connected in Databox |
| Correlation Agent | @n8n/n8n-nodes-langchain.agent | Combines website and paid analyses into final HTML report | Paid Acquisition Agent; OpenAI Chat Model 3 | Prepare Email | ## 2 - AI Agents + Databox MCP Setup<br>### What this section does<br>Three agents run in sequence:<br>- **Agent 1 - Website Analyst**: fetches sessions, bounce rate, pages per session, goal completions from your website analytics source<br>- **Agent 2 - Paid Acquisition Analyst**: fetches spend, CPC, CTR, ROAS and impressions for every connected paid platform<br>- **Agent 3 - Correlation Analyst**: receives both outputs, finds cross-channel patterns, ranks channel efficiency, and writes the final HTML report (no MCP calls needed)<br>### What you need to do<br>1. Click each **OpenAI Chat Model** node and add your API key<br>- You can replace with an **Anthropic Chat Model** node<br>2. Click each **Databox MCP Tool** node - set Authentication to `OAuth2` - authorize with your Databox account<br>3. Ensure at least one paid ads platform and a website analytics source are connected in Databox |
| OpenAI Chat Model 1 | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM backend for website analysis |  | Website Analysis Agent | ## 2 - AI Agents + Databox MCP Setup<br>### What this section does<br>Three agents run in sequence:<br>- **Agent 1 - Website Analyst**: fetches sessions, bounce rate, pages per session, goal completions from your website analytics source<br>- **Agent 2 - Paid Acquisition Analyst**: fetches spend, CPC, CTR, ROAS and impressions for every connected paid platform<br>- **Agent 3 - Correlation Analyst**: receives both outputs, finds cross-channel patterns, ranks channel efficiency, and writes the final HTML report (no MCP calls needed)<br>### What you need to do<br>1. Click each **OpenAI Chat Model** node and add your API key<br>- You can replace with an **Anthropic Chat Model** node<br>2. Click each **Databox MCP Tool** node - set Authentication to `OAuth2` - authorize with your Databox account<br>3. Ensure at least one paid ads platform and a website analytics source are connected in Databox |
| OpenAI Chat Model 2 | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM backend for paid acquisition analysis |  | Paid Acquisition Agent | ## 2 - AI Agents + Databox MCP Setup<br>### What this section does<br>Three agents run in sequence:<br>- **Agent 1 - Website Analyst**: fetches sessions, bounce rate, pages per session, goal completions from your website analytics source<br>- **Agent 2 - Paid Acquisition Analyst**: fetches spend, CPC, CTR, ROAS and impressions for every connected paid platform<br>- **Agent 3 - Correlation Analyst**: receives both outputs, finds cross-channel patterns, ranks channel efficiency, and writes the final HTML report (no MCP calls needed)<br>### What you need to do<br>1. Click each **OpenAI Chat Model** node and add your API key<br>- You can replace with an **Anthropic Chat Model** node<br>2. Click each **Databox MCP Tool** node - set Authentication to `OAuth2` - authorize with your Databox account<br>3. Ensure at least one paid ads platform and a website analytics source are connected in Databox |
| OpenAI Chat Model 3 | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM backend for final HTML synthesis |  | Correlation Agent | ## 2 - AI Agents + Databox MCP Setup<br>### What this section does<br>Three agents run in sequence:<br>- **Agent 1 - Website Analyst**: fetches sessions, bounce rate, pages per session, goal completions from your website analytics source<br>- **Agent 2 - Paid Acquisition Analyst**: fetches spend, CPC, CTR, ROAS and impressions for every connected paid platform<br>- **Agent 3 - Correlation Analyst**: receives both outputs, finds cross-channel patterns, ranks channel efficiency, and writes the final HTML report (no MCP calls needed)<br>### What you need to do<br>1. Click each **OpenAI Chat Model** node and add your API key<br>- You can replace with an **Anthropic Chat Model** node<br>2. Click each **Databox MCP Tool** node - set Authentication to `OAuth2` - authorize with your Databox account<br>3. Ensure at least one paid ads platform and a website analytics source are connected in Databox |
| Prepare Email | n8n-nodes-base.code | Builds email subject and body payload | Correlation Agent | Send Email | ## 3 - Email Report Output<br>### What this section does<br>Formats the correlation analysis as a styled HTML email and delivers it to the configured recipient every morning.<br>### What you need to do<br>- Click the **Send Email** node and add your **Gmail credential**<br>- Update the **To** field with the recipient email address<br>- **Optional**: Add a Slack node after Prepare Email for an additional Slack notification |
| Send Email | n8n-nodes-base.gmail | Sends final HTML email through Gmail | Prepare Email |  | ## 3 - Email Report Output<br>### What this section does<br>Formats the correlation analysis as a styled HTML email and delivers it to the configured recipient every morning.<br>### What you need to do<br>- Click the **Send Email** node and add your **Gmail credential**<br>- Update the **To** field with the recipient email address<br>- **Optional**: Add a Slack node after Prepare Email for an additional Slack notification |
| Sticky Note | n8n-nodes-base.stickyNote | Canvas documentation and workflow description |  |  | ## Daily Paid Acquisition Intelligence Report via Databox MCP<br>Get a daily intelligence briefing on your paid acquisition performance. Three specialized AI agents run in sequence: Agent 1 analyzes website behavior, Agent 2 analyzes paid channel performance across all connected platforms, and Agent 3 runs cross-channel correlation analysis - surfacing insights like cost per quality visit, channel efficiency rankings, and concrete budget reallocation recommendations.<br>### How it works<br>`Schedule Trigger` - `Agent 1: Website Analysis (Databox MCP)` - `Agent 2: Paid Acquisition Analysis (Databox MCP)` - `Agent 3: Correlations + Recommendations` - `HTML Email Report`<br>### What you need<br>- Databox account with website analytics and at least one paid ads platform connected - Free plan available: https://databox.com/?ref=n8n<br>- OpenAI API key<br>- Gmail account<br>### Supported Data Sources<br>Google Analytics - Google Ads - Facebook Ads - LinkedIn Ads - TikTok Ads - YouTube Ads - Microsoft Ads - Reddit Ads - And 100+ more - https://databox.com/integrations |
| Sticky Note 1 | n8n-nodes-base.stickyNote | Canvas documentation for schedule section |  |  | ## 1 - Schedule<br>### What this section does<br>Triggers the workflow every day at 8 AM and captures today's date so all three agents use consistent reporting windows.<br>### Change the schedule (optional)<br>- Click the "Every Monday 8 AM" node<br>- Click on the Cron Expression field<br>- Modify the schedule (e.g., change to daily, weekly on different day, or custom time) |
| Sticky Note 2 | n8n-nodes-base.stickyNote | Canvas documentation for AI and Databox setup |  |  | ## 2 - AI Agents + Databox MCP Setup<br>### What this section does<br>Three agents run in sequence:<br>- **Agent 1 - Website Analyst**: fetches sessions, bounce rate, pages per session, goal completions from your website analytics source<br>- **Agent 2 - Paid Acquisition Analyst**: fetches spend, CPC, CTR, ROAS and impressions for every connected paid platform<br>- **Agent 3 - Correlation Analyst**: receives both outputs, finds cross-channel patterns, ranks channel efficiency, and writes the final HTML report (no MCP calls needed)<br>### What you need to do<br>1. Click each **OpenAI Chat Model** node and add your API key<br>- You can replace with an **Anthropic Chat Model** node<br>2. Click each **Databox MCP Tool** node - set Authentication to `OAuth2` - authorize with your Databox account<br>3. Ensure at least one paid ads platform and a website analytics source are connected in Databox |
| Sticky Note 3 | n8n-nodes-base.stickyNote | Canvas documentation for email output section |  |  | ## 3 - Email Report Output<br>### What this section does<br>Formats the correlation analysis as a styled HTML email and delivers it to the configured recipient every morning.<br>### What you need to do<br>- Click the **Send Email** node and add your **Gmail credential**<br>- Update the **To** field with the recipient email address<br>- **Optional**: Add a Slack node after Prepare Email for an additional Slack notification |
| Sticky Note1 | n8n-nodes-base.stickyNote | Canvas note with Databox MCP connection resource |  |  | ### How to connect Databox MCP Tool in n8n<br>@[youtube](892KtXhv-vI) |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it something like: **Daily paid acquisition intelligence report**.

2. **Add a Schedule Trigger node**
   - Node type: **Schedule Trigger**
   - Rename it to **Every Day 8 AM**
   - Set the schedule rule to **Cron Expression**
   - Use: `0 8 * * *`
   - This makes the workflow run daily at 8:00 AM.

3. **Add a Date & Time node**
   - Node type: **Date & Time**
   - Rename it to **Get Current Date**
   - Leave default options unless you need a custom format or timezone
   - Connect:
     - **Every Day 8 AM → Get Current Date**

4. **Add the first OpenAI model node**
   - Node type: **OpenAI Chat Model**
   - Rename it to **OpenAI Chat Model 1**
   - Configure:
     - Model: **gpt-4o**
     - Max Tokens: **2048**
   - Add your **OpenAI API credential**

5. **Add the first Databox MCP tool node**
   - Node type: **MCP Client Tool**
   - Rename it to **Databox MCP Tool**
   - Configure:
     - Endpoint URL: `https://mcp.databox.com/mcp`
     - Authentication: **OAuth2**
   - Add/authorize a **Databox OAuth2 credential**

6. **Add the first AI Agent node**
   - Node type: **AI Agent**
   - Rename it to **Website Analysis Agent**
   - Set prompt type to **Define**
   - In the main prompt/text field, use logic equivalent to:
     - include today’s date from `{{ $json.formattedDate }}`
     - ask for website performance data for last 7 days vs previous 7 days
     - request traffic, engagement, and conversion analysis
   - In the system message, instruct the agent to:
     - call `list_accounts`
     - call `list_data_sources`
     - identify website analytics sources
     - fetch sessions/users, bounce rate, pages per session, average session duration, goal completions/conversions, conversion rate
     - compare last 7 days vs previous 7 days
     - calculate WoW changes
     - output structured plain text only
     - return `"No website analytics data available."` if nothing relevant is connected
   - Connect:
     - **Get Current Date → Website Analysis Agent** (main)
     - **OpenAI Chat Model 1 → Website Analysis Agent** (`ai_languageModel`)
     - **Databox MCP Tool → Website Analysis Agent** (`ai_tool`)

7. **Add the second OpenAI model node**
   - Node type: **OpenAI Chat Model**
   - Rename it to **OpenAI Chat Model 2**
   - Configure:
     - Model: **gpt-4o**
     - Max Tokens: **2048**
   - Use the same or another valid OpenAI credential

8. **Add the second Databox MCP tool node**
   - Node type: **MCP Client Tool**
   - Rename it to **Databox MCP Tool 2**
   - Configure:
     - Endpoint URL: `https://mcp.databox.com/mcp`
     - Authentication: **OAuth2**
   - Reuse the same Databox credential if desired

9. **Add the second AI Agent node**
   - Node type: **AI Agent**
   - Rename it to **Paid Acquisition Agent**
   - Set prompt type to **Define**
   - In the main prompt/text field:
     - include today’s date using `{{ $('Get Current Date').item.json.formattedDate }}`
     - include Agent 1 output using `{{ $json.output }}`
     - ask for paid acquisition data for the same periods
     - request spend efficiency, volume metrics, and conversion metrics across all connected paid platforms
   - In the system message, instruct the agent to:
     - call `list_accounts`
     - call `list_data_sources`
     - identify paid advertising platforms
     - fetch spend/cost, clicks, CPC, CTR, impressions, ROAS or conversions
     - calculate WoW changes per platform
     - calculate aggregate totals across platforms
     - output structured plain text only
     - return `"No paid advertising data available."` if none are connected
   - Connect:
     - **Website Analysis Agent → Paid Acquisition Agent** (main)
     - **OpenAI Chat Model 2 → Paid Acquisition Agent** (`ai_languageModel`)
     - **Databox MCP Tool 2 → Paid Acquisition Agent** (`ai_tool`)

10. **Add the third OpenAI model node**
    - Node type: **OpenAI Chat Model**
    - Rename it to **OpenAI Chat Model 3**
    - Configure:
      - Model: **gpt-4o**
      - Max Tokens: **4096**
    - Add your OpenAI credential

11. **Add the third AI Agent node**
    - Node type: **AI Agent**
    - Rename it to **Correlation Agent**
    - Set prompt type to **Define**
    - In the main prompt/text field:
      - include today’s date with `{{ $('Get Current Date').item.json.formattedDate }}`
      - include website analysis with `{{ $('Website Analysis Agent').item.json.output }}`
      - include paid analysis with `{{ $json.output }}`
      - instruct the model to produce an HTML email body containing:
        1. Executive Summary
        2. Website Performance table
        3. Paid Acquisition per-platform tables
        4. Correlation Analysis
        5. Exactly 3 Recommendations
        6. Footer
      - require output to start with a `<div>` tag and exclude `<html>`, `<head>`, and `<body>`
    - In the system message, specify formatting rules such as:
      - Databox blue `#3164FA`
      - green/red WoW colors
      - Helvetica Neue / Arial fonts
      - styled summary/recommendation boxes
      - comma-formatted numbers
      - `$X,XXX.XX` for currency
      - `X.XX%` for percentages
      - graceful handling of missing data
    - Connect:
      - **Paid Acquisition Agent → Correlation Agent** (main)
      - **OpenAI Chat Model 3 → Correlation Agent** (`ai_languageModel`)
    - Do **not** connect an MCP tool to this agent.

12. **Add a Code node**
    - Node type: **Code**
    - Rename it to **Prepare Email**
    - Use JavaScript similar to this logic:
      - read the incoming HTML from `json.output`
      - create a formatted date string
      - return:
        - `subject`: `Daily Paid Acquisition Report - <date>`
        - `htmlBody`: the HTML generated by the Correlation Agent
    - Equivalent logic:
      - `const agentOutput = $input.first().json.output || '';`
      - create formatted date using `new Date().toLocaleDateString('en-US', ...)`
      - return one item with `subject` and `htmlBody`
    - Connect:
      - **Correlation Agent → Prepare Email**

13. **Add a Gmail node**
    - Node type: **Gmail**
    - Rename it to **Send Email**
    - Operation: send email
    - Set:
      - **To**: your recipient address
      - **Subject**: `{{ $json.subject }}`
      - **Message**: `{{ $json.htmlBody }}`
    - Add a **Gmail OAuth2 credential**
    - Connect:
      - **Prepare Email → Send Email**

14. **Add credentials**
    - **OpenAI credential**
      - Required by all three OpenAI Chat Model nodes
    - **Databox OAuth2 credential**
      - Required by both MCP Client Tool nodes
    - **Gmail OAuth2 credential**
      - Required by the Send Email node

15. **Validate Databox prerequisites**
    - Ensure the Databox account has:
      - at least one website analytics source connected
      - at least one paid ads platform connected
    - Typical examples:
      - website: Google Analytics / GA4
      - paid: Google Ads, Facebook Ads, LinkedIn Ads, TikTok Ads, Microsoft Ads, etc.

16. **Test the workflow manually**
    - Run from **Get Current Date** or trigger the workflow manually
    - Verify:
      - Agent 1 returns website plain text
      - Agent 2 returns paid acquisition plain text
      - Agent 3 returns HTML beginning with `<div>`
      - Prepare Email outputs `subject` and `htmlBody`
      - Gmail sends successfully

17. **Replace placeholder email address**
    - Change `user@example.com` to the actual recipient.

18. **Activate the workflow**
    - The schedule trigger only runs automatically when the workflow is active.

## Rebuild notes and constraints

- The workflow has **one entry point**: **Every Day 8 AM**
- There are **no sub-workflows**
- There are **two tool-enabled agents** and **one synthesis-only agent**
- The final HTML body is generated by AI, so output quality depends on:
  - source data availability,
  - model consistency,
  - prompt adherence,
  - token budget.

## Recommended hardening improvements when rebuilding

If you want a more production-grade version, consider adding:

1. **Validation after each agent**
   - Add IF or Code nodes to detect empty outputs.

2. **Fallback handling**
   - If HTML does not start with `<div>`, wrap or sanitize it before sending.

3. **Timezone normalization**
   - Use one explicit timezone in both Schedule and Date generation.

4. **Error notifications**
   - Add Slack or email alerts on failure branches.

5. **Recipient parameterization**
   - Move the recipient address into an environment variable or workflow static data.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Databox account with website analytics and at least one paid ads platform connected is required. Free plan mentioned. | https://databox.com/?ref=n8n |
| Supported sources include Google Analytics, Google Ads, Facebook Ads, LinkedIn Ads, TikTok Ads, YouTube Ads, Microsoft Ads, Reddit Ads, and many more. | https://databox.com/integrations |
| Resource for connecting the Databox MCP Tool in n8n. | YouTube: `892KtXhv-vI` |
| Gmail credentials must be added and the recipient email must be updated from the placeholder value. | Applies to the Send Email node |
| Optional enhancement suggested in the workflow canvas: add a Slack node after Prepare Email for additional notifications. | Applies after Prepare Email |