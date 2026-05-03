Detect underpriced MLS properties with GPT and alert via Gmail and Slack

https://n8nworkflows.xyz/workflows/detect-underpriced-mls-properties-with-gpt-and-alert-via-gmail-and-slack-14469


# Detect underpriced MLS properties with GPT and alert via Gmail and Slack

# 1. Workflow Overview

This workflow automates the detection of potentially underpriced real estate properties by combining MLS listing data, recent sales data, and AI-based valuation analysis. It runs on a schedule, consolidates market inputs, uses a primary pricing agent plus a secondary market-research tool, and sends alerts through Gmail and Slack when opportunities are found.

Typical use cases include:
- Real estate investment sourcing
- Brokerage market monitoring
- Competitive pricing intelligence
- Automated lead generation for acquisition teams

## 1.1 Scheduled Execution and Runtime Configuration

The workflow starts on a recurring schedule and immediately passes through a configuration node intended to centralize runtime values, such as API endpoints, thresholds, dates, or filters.

## 1.2 Market Data Collection and Aggregation

Two HTTP Request nodes fetch MLS listings and recent comparable sales in parallel. Their outputs are merged into one dataset that feeds the AI analysis stage.

## 1.3 AI Pricing and Market Intelligence

A main LangChain agent performs pricing analysis using an OpenRouter-hosted chat model. It is supported by:
- a structured output parser for normalized pricing results
- a secondary AI tool agent for market research
- a second OpenRouter chat model
- another structured output parser for the research tool

This creates a two-layer analysis: direct valuation plus broader market context.

## 1.4 Conditional Opportunity Detection and Alerting

An IF node checks whether underpriced properties were identified. If so, the workflow sends notifications via Gmail and Slack.

## 1.5 Documentation and In-Canvas Guidance

Multiple sticky notes document the purpose, prerequisites, setup, benefits, and functional grouping of the workflow directly inside the canvas.

---

# 2. Block-by-Block Analysis

## Block 1 — Scheduled Trigger and Configuration

### Overview
This block initiates the workflow on a recurring basis and prepares shared configuration values for downstream nodes. It acts as the control layer for periodic execution.

### Nodes Involved
- Daily Pricing Update Schedule
- Workflow Configuration

### Node Details

#### 1. Daily Pricing Update Schedule
- **Type and technical role:** `n8n-nodes-base.scheduleTrigger`  
  Entry-point trigger node that launches the workflow on a time-based schedule.
- **Configuration choices:**  
  Uses an interval-based rule. In the exported JSON, the interval object is present but not fully specified, so the final run cadence must be completed or verified in n8n after import.
- **Key expressions or variables used:**  
  None shown.
- **Input and output connections:**  
  - No input; this is an entry node.
  - Outputs to **Workflow Configuration**.
- **Version-specific requirements:**  
  Type version `1.3`.
- **Edge cases or potential failure types:**  
  - Misconfigured interval/cron causing no execution
  - Timezone mismatch
  - Workflow inactive, so no scheduled runs occur until manually activated
- **Sub-workflow reference:**  
  None.

#### 2. Workflow Configuration
- **Type and technical role:** `n8n-nodes-base.set`  
  Used to define or normalize workflow-level variables before API calls.
- **Configuration choices:**  
  The node currently has no visible fields configured in the JSON. It likely serves as a placeholder for values such as:
  - MLS API base URL
  - recent-sales query window
  - underpricing threshold
  - target ZIP/city/region
  - execution metadata
- **Key expressions or variables used:**  
  None currently visible.
- **Input and output connections:**  
  - Input from **Daily Pricing Update Schedule**
  - Outputs in parallel to **Fetch MLS Data** and **Fetch Recent Sales Data**
- **Version-specific requirements:**  
  Type version `3.4`.
- **Edge cases or potential failure types:**  
  - Downstream expressions may fail if this node is expected to provide fields but does not
  - Empty configuration may require manual completion after import
- **Sub-workflow reference:**  
  None.

---

## Block 2 — Market Data Collection and Aggregation

### Overview
This block gathers listing and comparable-sales data from external APIs and combines the results into one stream for AI analysis. It is the core data-ingestion layer.

### Nodes Involved
- Fetch MLS Data
- Fetch Recent Sales Data
- Combine All Market Data

### Node Details

#### 3. Fetch MLS Data
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Calls an MLS or listing provider API to retrieve active property listing data.
- **Configuration choices:**  
  The JSON does not expose request details such as URL, auth method, headers, query params, or pagination. This means the node must be configured manually with the MLS provider endpoint and credentials.
- **Key expressions or variables used:**  
  None visible, though it likely should reference values from **Workflow Configuration**.
- **Input and output connections:**  
  - Input from **Workflow Configuration**
  - Output to **Combine All Market Data** input 0
- **Version-specific requirements:**  
  Type version `4.3`.
- **Edge cases or potential failure types:**  
  - Authentication failure
  - Rate limiting
  - Network timeout
  - Unexpected response schema
  - Pagination not handled, resulting in incomplete listing coverage
  - Empty response for selected market/date filters
- **Sub-workflow reference:**  
  None.

#### 4. Fetch Recent Sales Data
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Retrieves comparable or recent sold-property data from an MLS or market data source.
- **Configuration choices:**  
  Request specifics are not present in the JSON. This node must be manually configured with endpoint, authentication, date filters, and any required geographic or property filters.
- **Key expressions or variables used:**  
  None visible.
- **Input and output connections:**  
  - Input from **Workflow Configuration**
  - Output to **Combine All Market Data** input 1
- **Version-specific requirements:**  
  Type version `4.3`.
- **Edge cases or potential failure types:**  
  - Invalid auth/token expiration
  - Query window too narrow or too broad
  - Data format mismatch relative to MLS listings
  - Missing sale dates/prices affecting valuation quality
- **Sub-workflow reference:**  
  None.

#### 5. Combine All Market Data
- **Type and technical role:** `n8n-nodes-base.merge`  
  Merges the two upstream datasets so the analysis stage can work with both listings and recent sales context.
- **Configuration choices:**  
  No explicit merge mode is shown in the JSON, so it uses default behavior for this node version. This is important because merge behavior in n8n depends heavily on mode:
  - append
  - combine by position
  - combine by matching fields
  - wait-for-both semantics
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  - Input 0 from **Fetch MLS Data**
  - Input 1 from **Fetch Recent Sales Data**
  - Output to **Pricing Analysis Agent**
- **Version-specific requirements:**  
  Type version `3.2`.
- **Edge cases or potential failure types:**  
  - Incorrect merge mode leading to malformed combined records
  - Uneven item counts between listings and sales inputs
  - One branch returns no data, blocking or degrading downstream analysis depending on mode
  - Schema collisions where similarly named fields overwrite each other
- **Sub-workflow reference:**  
  None.

---

## Block 3 — AI Pricing and Market Intelligence

### Overview
This block performs the core valuation logic. A primary AI agent analyzes pricing, while a secondary AI tool provides market research context. Both use structured output parsers to constrain model output into machine-usable form.

### Nodes Involved
- Pricing Analysis Agent
- Pricing Output Parser
- Market Research Agent Tool
- Research Output Parser
- OpenRouter Chat Model
- OpenRouter Chat Model1

### Node Details

#### 6. Pricing Analysis Agent
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent`  
  Main AI agent responsible for evaluating whether properties appear underpriced.
- **Configuration choices:**  
  - Has structured output parsing enabled
  - Uses a connected language model node: **OpenRouter Chat Model**
  - Has access to a tool node: **Market Research Agent Tool**
  - Receives merged market data from the previous block
- **Key expressions or variables used:**  
  Not visible in the JSON. In a functional build, this node should include prompts/instructions such as:
  - compare listing price vs recent sales
  - estimate fair market value
  - identify underpricing percentage
  - return only structured output
- **Input and output connections:**  
  - Main input from **Combine All Market Data**
  - AI language model input from **OpenRouter Chat Model**
  - AI output parser input from **Pricing Output Parser**
  - AI tool input from **Market Research Agent Tool**
  - Main output to **Check for Underpriced Properties**
- **Version-specific requirements:**  
  Type version `3.1`.
- **Edge cases or potential failure types:**  
  - Missing prompt instructions causing weak or generic analysis
  - Model output not matching parser schema
  - Tool invocation failure
  - Token/context overflow if raw MLS data is too large
  - Hallucinated valuations if comparable data is poor
- **Sub-workflow reference:**  
  None.

#### 7. Pricing Output Parser
- **Type and technical role:** `@n8n/n8n-nodes-langchain.outputParserStructured`  
  Enforces a structured schema on the pricing agent’s output.
- **Configuration choices:**  
  The exact schema is not visible. It should define fields such as:
  - property identifier
  - estimated market value
  - listing price
  - underpriced boolean
  - confidence score
  - rationale
- **Key expressions or variables used:**  
  None visible.
- **Input and output connections:**  
  - Connected to **Pricing Analysis Agent** as AI output parser
- **Version-specific requirements:**  
  Type version `1.3`.
- **Edge cases or potential failure types:**  
  - Parser schema too strict for actual model output
  - Missing required fields
  - Type mismatches, e.g. number returned as text
- **Sub-workflow reference:**  
  None.

#### 8. Market Research Agent Tool
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agentTool`  
  Tool-style AI agent callable by the main pricing agent to provide supporting market context and trend interpretation.
- **Configuration choices:**  
  - Has structured output parsing enabled
  - Uses a dedicated language model: **OpenRouter Chat Model1**
  - Returns its output back as a tool to the main agent
- **Key expressions or variables used:**  
  None visible. Typical tool prompt content would include:
  - summarize neighborhood or market trends
  - explain whether recent sales support a discount opportunity
  - provide caution flags such as low liquidity or declining prices
- **Input and output connections:**  
  - AI language model input from **OpenRouter Chat Model1**
  - AI output parser input from **Research Output Parser**
  - AI tool output to **Pricing Analysis Agent**
- **Version-specific requirements:**  
  Type version `3`.
- **Edge cases or potential failure types:**  
  - Tool result not aligned with primary agent schema
  - Recursive or unnecessary tool calls increasing latency/cost
  - Same token-limit issues as the main agent if too much data is passed
- **Sub-workflow reference:**  
  None.

#### 9. Research Output Parser
- **Type and technical role:** `@n8n/n8n-nodes-langchain.outputParserStructured`  
  Structures the result of the market research tool.
- **Configuration choices:**  
  Schema is not shown, but likely intended to normalize fields such as:
  - trend direction
  - comparable-sales signal
  - neighborhood commentary
  - risk notes
- **Key expressions or variables used:**  
  None visible.
- **Input and output connections:**  
  - Connected to **Market Research Agent Tool** as AI output parser
- **Version-specific requirements:**  
  Type version `1.3`.
- **Edge cases or potential failure types:**  
  - Invalid structured response from model
  - Missing fields required by the main agent
- **Sub-workflow reference:**  
  None.

#### 10. OpenRouter Chat Model
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenRouter`  
  Supplies the LLM used by the main pricing agent.
- **Configuration choices:**  
  - Model: `openai/gpt-5.2-pro`
  - Uses OpenRouter API credentials
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  - AI language model output to **Pricing Analysis Agent**
- **Version-specific requirements:**  
  Type version `1`.
- **Edge cases or potential failure types:**  
  - Credential failure at OpenRouter
  - Model availability changes
  - Cost spikes on large prompts
  - Vendor-side latency
- **Sub-workflow reference:**  
  None.

#### 11. OpenRouter Chat Model1
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenRouter`  
  Supplies the LLM used by the market research tool agent.
- **Configuration choices:**  
  - Model: `openai/gpt-5.2-pro`
  - Reuses the same OpenRouter credential set
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  - AI language model output to **Market Research Agent Tool**
- **Version-specific requirements:**  
  Type version `1`.
- **Edge cases or potential failure types:**  
  - Same as the main model node
  - Additional cost due to dual-agent design
- **Sub-workflow reference:**  
  None.

---

## Block 4 — Conditional Detection and Alert Delivery

### Overview
This block checks the analysis result and, when underpriced opportunities exist, sends notifications to email and Slack recipients. It is the operational output layer of the workflow.

### Nodes Involved
- Check for Underpriced Properties
- Send Underpriced Alert Email
- Send Slack Alert

### Node Details

#### 12. Check for Underpriced Properties
- **Type and technical role:** `n8n-nodes-base.if`  
  Branches execution based on whether the AI output indicates underpriced properties were found.
- **Configuration choices:**  
  The actual condition is not exposed in the JSON. This must be manually defined based on the pricing parser output, for example:
  - `underpriced = true`
  - `underpricedCount > 0`
  - `discountPercent >= threshold`
- **Key expressions or variables used:**  
  Not visible, but likely based on parsed AI output fields.
- **Input and output connections:**  
  - Input from **Pricing Analysis Agent**
  - The configured true branch feeds both **Send Underpriced Alert Email** and **Send Slack Alert**
- **Version-specific requirements:**  
  Type version `2.3`.
- **Edge cases or potential failure types:**  
  - Condition references a non-existent field
  - Parser output structure differs from IF expectations
  - False negatives if output is nested and not properly referenced
- **Sub-workflow reference:**  
  None.

#### 13. Send Underpriced Alert Email
- **Type and technical role:** `n8n-nodes-base.gmail`  
  Sends alert emails to one or more recipients.
- **Configuration choices:**  
  - Uses Gmail OAuth2 credential
  - Operation details such as recipient, subject, HTML/text body are not visible and must be configured
- **Key expressions or variables used:**  
  Not visible. Typical dynamic fields would include:
  - property address
  - listing price
  - estimated value
  - discount %
  - rationale and link to listing
- **Input and output connections:**  
  - Input from **Check for Underpriced Properties**
- **Version-specific requirements:**  
  Type version `2.2`.
- **Edge cases or potential failure types:**  
  - Gmail OAuth token expiration or missing consent scopes
  - Sending quota limits
  - Invalid recipient formatting
  - Message body referencing absent fields
- **Sub-workflow reference:**  
  None.

#### 14. Send Slack Alert
- **Type and technical role:** `n8n-nodes-base.slack`  
  Sends a notification message to a Slack workspace/channel.
- **Configuration choices:**  
  - Authentication: OAuth2
  - Uses Slack OAuth2 API credential
  - Specific channel, message content, and formatting are not visible
- **Key expressions or variables used:**  
  Not visible. Typical dynamic content would mirror the email summary.
- **Input and output connections:**  
  - Input from **Check for Underpriced Properties**
- **Version-specific requirements:**  
  Type version `2.4`.
- **Edge cases or potential failure types:**  
  - Slack OAuth scope mismatch
  - Wrong channel ID or inaccessible private channel
  - Message formatting errors
  - Rate limiting on high alert volumes
- **Sub-workflow reference:**  
  None.

---

## Block 5 — Canvas Documentation and Embedded Operational Notes

### Overview
This block contains only sticky notes. These do not affect execution, but they are important for maintainability, onboarding, and reconstruction.

### Nodes Involved
- Sticky Note
- Sticky Note1
- Sticky Note2
- Sticky Note3
- Sticky Note4
- Sticky Note5
- Sticky Note6

### Node Details

#### 15. Sticky Note
- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Documents the alerting section.
- **Configuration choices:**  
  Text content explains Gmail/Slack alert distribution.
- **Input and output connections:**  
  None.
- **Version-specific requirements:**  
  Type version `1`.
- **Edge cases or potential failure types:**  
  None operational.
- **Sub-workflow reference:**  
  None.

#### 16. Sticky Note1
- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Provides the high-level workflow explanation and intended users.
- **Configuration choices:**  
  Rich descriptive text summarizing the complete workflow.
- **Input and output connections:**  
  None.
- **Version-specific requirements:**  
  Type version `1`.
- **Edge cases or potential failure types:**  
  None operational.
- **Sub-workflow reference:**  
  None.

#### 17. Sticky Note2
- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Captures prerequisites, use cases, customization guidance, and benefits.
- **Configuration choices:**  
  Includes setup assumptions and strategic value notes.
- **Input and output connections:**  
  None.
- **Version-specific requirements:**  
  Type version `1`.
- **Edge cases or potential failure types:**  
  None operational.
- **Sub-workflow reference:**  
  None.

#### 18. Sticky Note3
- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Provides setup instructions for credentials and schedule.
- **Configuration choices:**  
  Includes references to OpenAI in the note text, although the actual workflow uses OpenRouter model nodes.
- **Input and output connections:**  
  None.
- **Version-specific requirements:**  
  Type version `1`.
- **Edge cases or potential failure types:**  
  Possible confusion because the note mentions:
  - OpenAI API key
  - Gmail SMTP
  - Slack webhook URL  
  while the actual nodes use:
  - OpenRouter credentials
  - Gmail OAuth2
  - Slack OAuth2
- **Sub-workflow reference:**  
  None.

#### 19. Sticky Note4
- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Documents the data aggregation block.
- **Configuration choices:**  
  Short summary of multi-source market coverage.
- **Input and output connections:**  
  None.
- **Version-specific requirements:**  
  Type version `1`.
- **Edge cases or potential failure types:**  
  None operational.
- **Sub-workflow reference:**  
  None.

#### 20. Sticky Note5
- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Documents the AI dual-analysis block.
- **Configuration choices:**  
  Describes pricing and market-research AI behavior.
- **Input and output connections:**  
  None.
- **Version-specific requirements:**  
  Type version `1`.
- **Edge cases or potential failure types:**  
  None operational.
- **Sub-workflow reference:**  
  None.

#### 21. Sticky Note6
- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Explains the rationale for using AI for scalable analyst-style insights.
- **Configuration choices:**  
  Conceptual note for maintainers and users.
- **Input and output connections:**  
  None.
- **Version-specific requirements:**  
  Type version `1`.
- **Edge cases or potential failure types:**  
  None operational.
- **Sub-workflow reference:**  
  None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Daily Pricing Update Schedule | Schedule Trigger | Starts the workflow on a recurring schedule |  | Workflow Configuration | ## How It Works\nThis workflow automates competitive real estate pricing analysis by combining multiple MLS data sources with AI-powered market intelligence. Designed for real estate professionals, property managers, and investment analysts, it solves the critical challenge of identifying underpriced properties in competitive markets where manual analysis is time-consuming and prone to oversight. The system fetches listings from multiple MLS platforms, consolidates market data, and deploys specialized AI agents for dual-layer analysis. The Pricing Agent evaluates individual property valuations against market comparables, while the Market Research Agent provides broader market context and trend insights. When underpriced opportunities are detected, automated alerts are dispatched via email and Slack, enabling rapid response to market opportunities. Operating on a daily schedule, this workflow transforms hours of manual research into automated intelligence delivery. |
| Workflow Configuration | Set | Defines shared runtime values before API requests | Daily Pricing Update Schedule | Fetch MLS Data; Fetch Recent Sales Data | ## How It Works\nThis workflow automates competitive real estate pricing analysis by combining multiple MLS data sources with AI-powered market intelligence. Designed for real estate professionals, property managers, and investment analysts, it solves the critical challenge of identifying underpriced properties in competitive markets where manual analysis is time-consuming and prone to oversight. The system fetches listings from multiple MLS platforms, consolidates market data, and deploys specialized AI agents for dual-layer analysis. The Pricing Agent evaluates individual property valuations against market comparables, while the Market Research Agent provides broader market context and trend insights. When underpriced opportunities are detected, automated alerts are dispatched via email and Slack, enabling rapid response to market opportunities. Operating on a daily schedule, this workflow transforms hours of manual research into automated intelligence delivery. |
| Fetch MLS Data | HTTP Request | Retrieves MLS listing data | Workflow Configuration | Combine All Market Data | ## Multi-Source Data Aggregation\nFetches and combines MLS listings from multiple platforms to create comprehensive market coverage. |
| Fetch Recent Sales Data | HTTP Request | Retrieves recent comparable sales data | Workflow Configuration | Combine All Market Data | ## Multi-Source Data Aggregation\nFetches and combines MLS listings from multiple platforms to create comprehensive market coverage. |
| Combine All Market Data | Merge | Combines listing and sales data for analysis | Fetch MLS Data; Fetch Recent Sales Data | Pricing Analysis Agent | ## Multi-Source Data Aggregation\nFetches and combines MLS listings from multiple platforms to create comprehensive market coverage. |
| Pricing Analysis Agent | LangChain Agent | Main AI agent for underpricing evaluation | Combine All Market Data; OpenRouter Chat Model; Pricing Output Parser; Market Research Agent Tool | Check for Underpriced Properties | ## What: AI-Powered Dual Analysis\nDeploys specialized OpenAI agents for pricing evaluation and market research with structured output parsing. |
| Pricing Output Parser | Structured Output Parser | Constrains pricing-agent output into structured schema |  | Pricing Analysis Agent | ## What: AI-Powered Dual Analysis\nDeploys specialized OpenAI agents for pricing evaluation and market research with structured output parsing. |
| Market Research Agent Tool | LangChain Agent Tool | Provides market-context research callable by the main agent | OpenRouter Chat Model1; Research Output Parser | Pricing Analysis Agent | ## Why: Expert-Level Insights\nAI agents replicate analyst expertise at scale, providing consistent, objective valuations and contextual market intelligence simultaneously. |
| Research Output Parser | Structured Output Parser | Constrains research-tool output into structured schema |  | Market Research Agent Tool | ## Why: Expert-Level Insights\nAI agents replicate analyst expertise at scale, providing consistent, objective valuations and contextual market intelligence simultaneously. |
| Check for Underpriced Properties | IF | Filters execution to alert only when opportunities are found | Pricing Analysis Agent | Send Underpriced Alert Email; Send Slack Alert | ## Automated Alert Distribution\nSends formatted notifications through Gmail and Slack when underpriced properties are identified. |
| Send Underpriced Alert Email | Gmail | Sends email notifications for detected opportunities | Check for Underpriced Properties |  | ## Automated Alert Distribution\nSends formatted notifications through Gmail and Slack when underpriced properties are identified. |
| Send Slack Alert | Slack | Sends Slack notifications for detected opportunities | Check for Underpriced Properties |  | ## Automated Alert Distribution\nSends formatted notifications through Gmail and Slack when underpriced properties are identified. |
| Sticky Note | Sticky Note | Canvas documentation for alerting block |  |  | ## Automated Alert Distribution\nSends formatted notifications through Gmail and Slack when underpriced properties are identified. |
| Sticky Note1 | Sticky Note | High-level workflow documentation |  |  | ## How It Works\nThis workflow automates competitive real estate pricing analysis by combining multiple MLS data sources with AI-powered market intelligence. Designed for real estate professionals, property managers, and investment analysts, it solves the critical challenge of identifying underpriced properties in competitive markets where manual analysis is time-consuming and prone to oversight. The system fetches listings from multiple MLS platforms, consolidates market data, and deploys specialized AI agents for dual-layer analysis. The Pricing Agent evaluates individual property valuations against market comparables, while the Market Research Agent provides broader market context and trend insights. When underpriced opportunities are detected, automated alerts are dispatched via email and Slack, enabling rapid response to market opportunities. Operating on a daily schedule, this workflow transforms hours of manual research into automated intelligence delivery. |
| Sticky Note2 | Sticky Note | Documents prerequisites, use cases, customization, and benefits |  |  | ## Prerequisites\nOpenAI API account with GPT-4 access, MLS data provider API credentials\n## Use Cases\nInvestment firms identifying acquisition targets, real estate brokerages monitoring competitive listings\n## Customization\nModify AI agent prompts for specific property types, adjust underpricing threshold percentages\n## Benefits\nReduces manual research time by 90%, eliminates human bias in valuation analysis |
| Sticky Note3 | Sticky Note | Setup guidance note |  |  | ## Setup Steps\n1. Configure MLS API credentials in "Fetch MLS Data" and "Fetch Recent Sales Data" nodes\n2. Add OpenAI API key in "OpenAI Model - Pricing Agent"\n3. Set Gmail SMTP credentials in "Send Underpriced Alert Email" node with recipient addresses\n4. Configure Slack webhook URL in "Send Slack Alert" node for channel notifications\n5. Adjust "Daily Pricing Update Schedule" cron expression for preferred execution time |
| Sticky Note4 | Sticky Note | Canvas documentation for data ingestion block |  |  | ## Multi-Source Data Aggregation\nFetches and combines MLS listings from multiple platforms to create comprehensive market coverage. |
| Sticky Note5 | Sticky Note | Canvas documentation for AI pricing block |  |  | ## What: AI-Powered Dual Analysis\nDeploys specialized OpenAI agents for pricing evaluation and market research with structured output parsing. |
| Sticky Note6 | Sticky Note | Canvas documentation for market insight block |  |  | ## Why: Expert-Level Insights\nAI agents replicate analyst expertise at scale, providing consistent, objective valuations and contextual market intelligence simultaneously. |
| OpenRouter Chat Model | OpenRouter Chat Model | LLM backend for the pricing agent |  | Pricing Analysis Agent | ## What: AI-Powered Dual Analysis\nDeploys specialized OpenAI agents for pricing evaluation and market research with structured output parsing. |
| OpenRouter Chat Model1 | OpenRouter Chat Model | LLM backend for the research tool agent |  | Market Research Agent Tool | ## Why: Expert-Level Insights\nAI agents replicate analyst expertise at scale, providing consistent, objective valuations and contextual market intelligence simultaneously. |

---

# 4. Reproducing the Workflow from Scratch

Below is a practical rebuild sequence in n8n. Because several node parameters are blank in the JSON export, this section includes the missing setup decisions you must define manually.

## Step 1 — Create the schedule trigger
1. Add a **Schedule Trigger** node.
2. Name it **Daily Pricing Update Schedule**.
3. Configure it to run once per day, or set your preferred interval/cron expression.
4. Confirm the workflow timezone in n8n settings if alert timing matters.

## Step 2 — Add a configuration node
5. Add a **Set** node.
6. Name it **Workflow Configuration**.
7. Connect **Daily Pricing Update Schedule → Workflow Configuration**.
8. Add fields you want to reuse downstream, for example:
   - `mlsApiBaseUrl`
   - `salesApiBaseUrl`
   - `targetMarket`
   - `lookbackDays`
   - `underpricingThresholdPct`
   - `maxRecords`
9. If desired, keep raw input plus added fields so downstream expressions can reference them.

## Step 3 — Add the MLS listings request
10. Add an **HTTP Request** node.
11. Name it **Fetch MLS Data**.
12. Connect **Workflow Configuration → Fetch MLS Data**.
13. Configure:
   - Method: usually `GET`
   - URL: your MLS/provider listings endpoint
   - Authentication: per provider requirements
   - Query parameters: market, status=active, property type, limit, etc.
14. If using values from the Set node, reference them with expressions such as:
   - `{{$json.mlsApiBaseUrl}}/...`
   - `{{$json.targetMarket}}`
15. Enable pagination if your provider returns paginated results.

## Step 4 — Add the recent sales request
16. Add another **HTTP Request** node.
17. Name it **Fetch Recent Sales Data**.
18. Connect **Workflow Configuration → Fetch Recent Sales Data**.
19. Configure:
   - Method: usually `GET`
   - URL: recent sales/comparables endpoint
   - Authentication: same or separate provider credentials
   - Query parameters: market, sold date range, property type, limit
20. Use expressions from the configuration node for date windows and location filters.

## Step 5 — Merge the two data sources
21. Add a **Merge** node.
22. Name it **Combine All Market Data**.
23. Connect:
   - **Fetch MLS Data → Combine All Market Data** input 1
   - **Fetch Recent Sales Data → Combine All Market Data** input 2
24. Choose the merge mode based on your data shape:
   - **Append** if you just want both datasets passed together
   - **Combine by position** if both lists align item-to-item
   - **Combine by matching fields** if both datasets share keys such as ZIP, MLS ID, or address
25. Test this node carefully, because merge behavior determines whether the AI receives sensible inputs.

## Step 6 — Add the primary model for pricing analysis
26. Add an **OpenRouter Chat Model** node.
27. Name it **OpenRouter Chat Model**.
28. Set the model to **openai/gpt-5.2-pro**.
29. Create or select an **OpenRouter API credential**.
30. Keep default options unless you need temperature or response adjustments.

## Step 7 — Add the pricing output parser
31. Add a **Structured Output Parser** node from the LangChain nodes.
32. Name it **Pricing Output Parser**.
33. Define a strict structured schema. A practical schema would include fields like:
   - `underpricedProperties` (array)
   - `propertyId` (string)
   - `address` (string)
   - `listingPrice` (number)
   - `estimatedMarketValue` (number)
   - `discountPercent` (number)
   - `underpriced` (boolean)
   - `confidence` (number)
   - `reasoning` (string)
34. Keep field names consistent with what the IF and alert nodes will reference later.

## Step 8 — Add the market research model
35. Add another **OpenRouter Chat Model** node.
36. Name it **OpenRouter Chat Model1**.
37. Set the model to **openai/gpt-5.2-pro**.
38. Reuse the same OpenRouter credential or a different one if needed for billing separation.

## Step 9 — Add the research output parser
39. Add another **Structured Output Parser** node.
40. Name it **Research Output Parser**.
41. Define a schema for market-context output, for example:
   - `marketTrend` (string)
   - `supportingComps` (array)
   - `riskFlags` (array)
   - `summary` (string)

## Step 10 — Add the market research tool agent
42. Add a **LangChain Agent Tool** node.
43. Name it **Market Research Agent Tool**.
44. Enable structured output parsing.
45. Connect:
   - **OpenRouter Chat Model1 → Market Research Agent Tool** via AI language model connection
   - **Research Output Parser → Market Research Agent Tool** via AI output parser connection
46. Add instructions telling the tool to provide market-level context, not final alert decisions. Example responsibilities:
   - assess trend direction
   - summarize comparable-sales context
   - identify caution flags
   - return structured output only

## Step 11 — Add the main pricing agent
47. Add a **LangChain Agent** node.
48. Name it **Pricing Analysis Agent**.
49. Enable structured output parsing.
50. Connect:
   - **Combine All Market Data → Pricing Analysis Agent** via main connection
   - **OpenRouter Chat Model → Pricing Analysis Agent** via AI language model connection
   - **Pricing Output Parser → Pricing Analysis Agent** via AI output parser connection
   - **Market Research Agent Tool → Pricing Analysis Agent** via AI tool connection
51. Configure the agent prompt to:
   - review active listings against recent sales
   - call the market research tool when broader context is needed
   - identify underpriced listings using your threshold
   - output only the parser-defined structured schema
52. Include your threshold explicitly, e.g. “flag a property as underpriced when listing price is at least 8% below estimated value.”

## Step 12 — Add the decision node
53. Add an **IF** node.
54. Name it **Check for Underpriced Properties**.
55. Connect **Pricing Analysis Agent → Check for Underpriced Properties**.
56. Define the condition based on your parser schema. Good examples:
   - `{{$json.underpricedProperties.length > 0}}`
   - or `{{$json.underpriced === true}}`
57. Ensure the node is checking the exact field path returned by the parser.

## Step 13 — Add Gmail alerting
58. Add a **Gmail** node.
59. Name it **Send Underpriced Alert Email**.
60. Connect the **true** output of **Check for Underpriced Properties** to this node.
61. Create/select a **Gmail OAuth2 credential**.
62. Configure:
   - Operation: send email
   - To: your target recipients
   - Subject: dynamic alert subject, e.g. `Underpriced MLS opportunities detected`
   - Body: include address, listing price, estimated value, discount %, confidence, and reasoning
63. If multiple properties are returned, consider formatting the body as HTML or a bullet list.

## Step 14 — Add Slack alerting
64. Add a **Slack** node.
65. Name it **Send Slack Alert**.
66. Connect the **true** output of **Check for Underpriced Properties** to this node as well.
67. Create/select a **Slack OAuth2 API credential**.
68. Configure:
   - Authentication: OAuth2
   - Operation: send message
   - Channel: target Slack channel
   - Message text: concise summary of the detected opportunities
69. Optionally include structured blocks or links to the underlying listings.

## Step 15 — Add optional canvas notes
70. Add **Sticky Note** nodes if you want the same visual documentation as the original workflow.
71. Suggested note sections:
   - workflow purpose
   - prerequisites
   - setup steps
   - data aggregation
   - AI analysis
   - alerting

## Step 16 — Test and validate
72. Run the workflow manually with a narrow dataset first.
73. Verify both HTTP nodes return the fields the agent prompt expects.
74. Validate the merge output shape before testing AI nodes.
75. Confirm the parser schemas match actual model output.
76. Check that the IF condition references the parsed result correctly.
77. Send test email and Slack messages.
78. Activate the workflow only after end-to-end validation.

## Credential setup requirements

### OpenRouter
- Required for **OpenRouter Chat Model** and **OpenRouter Chat Model1**
- Must have access to the configured model `openai/gpt-5.2-pro`
- Watch for model naming changes or provider restrictions

### Gmail OAuth2
- Required for **Send Underpriced Alert Email**
- Ensure the connected Google account has permission to send mail
- Reauthorize if tokens expire

### Slack OAuth2 API
- Required for **Send Slack Alert**
- Ensure the app has permission to post to the selected channel
- Private channels require the app to be explicitly added

### MLS / Sales API authentication
- Required for **Fetch MLS Data** and **Fetch Recent Sales Data**
- May be header-based API key, bearer token, or OAuth2 depending on provider
- If the same provider serves both endpoints, you can often reuse one credential definition

## Important implementation note
The exported JSON omits the detailed prompts, parser schemas, HTTP request URLs, and IF conditions. Those missing pieces are not optional: they must be explicitly designed during reconstruction. The workflow structure is complete, but several operational details are placeholders.

## Sub-workflow setup
This workflow does **not** contain any Execute Workflow or sub-workflow invocation nodes. No sub-workflow needs to be created.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| OpenAI-related sticky note content does not fully match the actual node implementation. The workflow uses OpenRouter model nodes rather than direct OpenAI model nodes. | Applies to setup documentation inside the canvas |
| The setup sticky note mentions Gmail SMTP and Slack webhook configuration, but the actual nodes use Gmail OAuth2 and Slack OAuth2. | Credential alignment note |
| Workflow status is currently inactive. Scheduled runs will not occur until the workflow is activated in n8n. | Operational note |
| The Schedule Trigger interval is not fully explicit in the exported JSON and should be verified after import. | Trigger validation note |
| The merge node’s default mode is not visible in the export; verify it carefully because it affects data integrity for the AI analysis stage. | Data-shaping note |
| The workflow relies on structured parsing for both AI layers. Parser schema design is critical for reliable downstream conditions and alerts. | AI implementation note |