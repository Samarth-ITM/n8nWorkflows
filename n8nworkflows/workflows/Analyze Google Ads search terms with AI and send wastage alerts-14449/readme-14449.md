Analyze Google Ads search terms with AI and send wastage alerts

https://n8nworkflows.xyz/workflows/analyze-google-ads-search-terms-with-ai-and-send-wastage-alerts-14449


# Analyze Google Ads search terms with AI and send wastage alerts

# 1. Workflow Overview

This workflow automates daily Google Ads search term analysis for brand/search campaigns, uses AI to identify likely wasted spend from non-brand queries, and distributes a structured alert report to multiple communication channels.

Its main use case is paid search monitoring: every day at 8 AM, it fetches Google Ads campaigns, narrows them to active search campaigns matching a target naming convention, pulls recent search term performance, removes obvious brand/excluded terms, aggregates metrics, submits the data to an AI agent, then formats and sends the report to Slack, Microsoft Teams, Discord, and WhatsApp via Rapiwa.

## 1.1 Scheduled Trigger and Campaign Discovery
This block starts the workflow on a schedule and retrieves Google Ads campaigns from a specified manager/client account combination.

## 1.2 Campaign Filtering and Search Term Retrieval
This block keeps only enabled campaigns matching a specific search campaign naming pattern, then queries the Google Ads API for search-term-level performance data.

## 1.3 Data Cleaning, Filtering, and Aggregation
This block normalizes the raw API response, calculates marketing metrics, removes brand and excluded terms, and aggregates duplicate search term rows across dates.

## 1.4 AI Analysis Setup and Execution
This block packages the prepared dataset with brand context, routes it through a selectable LLM configuration, enforces a structured JSON output format, and runs the AI analysis.

## 1.5 Report Formatting and Distribution
This block converts the AI output into a block-based report payload and sends it to multiple messaging platforms.

---

# 2. Block-by-Block Analysis

## 2.1 Scheduled Trigger and Campaign Discovery

### Overview
This block initiates the workflow every day at 8 AM and fetches Google Ads campaigns for a configured client account under a manager account. It is the entry point for the entire automation.

### Nodes Involved
- Schedule Trigger (Start Everyday at 8AM)
- Fetch Google Campaigns

### Node Details

#### Schedule Trigger (Start Everyday at 8AM)
- **Type and role:** `n8n-nodes-base.scheduleTrigger`; workflow entry node that runs on a daily schedule.
- **Configuration choices:** Configured with an hourly trigger rule set to run at hour `8`.
- **Key expressions or variables used:** None.
- **Input and output connections:** No input; outputs to **Fetch Google Campaigns**.
- **Version-specific requirements:** Type version `1.3`.
- **Edge cases / failures:**
  - Workflow must be active for the trigger to fire.
  - Server timezone affects the exact execution time unless instance timezone is separately controlled.
  - If n8n is down at trigger time, the run may be missed depending on deployment setup.
- **Sub-workflow reference:** None.

#### Fetch Google Campaigns
- **Type and role:** `n8n-nodes-base.googleAds`; retrieves campaigns from Google Ads.
- **Configuration choices:**
  - Uses Google Ads credentials.
  - `clientCustomerId` is a placeholder: `add_your_client_ID`.
  - `managerCustomerId` is a placeholder: `add_your_manager_ID`.
  - `additionalOptions.dateRange` is set to `TODAY`, likely only to satisfy node/API behavior while fetching campaigns.
- **Key expressions or variables used:** Static placeholders only.
- **Input and output connections:** Input from **Schedule Trigger (Start Everyday at 8AM)**; output to **Get Only 'Enabled/Active' Campaigns**.
- **Version-specific requirements:** Type version `1`.
- **Edge cases / failures:**
  - OAuth2 credential issues.
  - Invalid manager/client account relationship.
  - Missing Google Ads API access or insufficient permissions.
  - Placeholder IDs must be replaced with actual numeric customer IDs.
- **Sub-workflow reference:** None.

---

## 2.2 Campaign Filtering and Search Term Retrieval

### Overview
This block narrows the campaign list to enabled campaigns and then to a specific target search campaign naming pattern. For each matching campaign, it performs a direct Google Ads API query to fetch recent search term performance.

### Nodes Involved
- Get Only 'Enabled/Active' Campaigns
- Filtering For A Specific Search Google Campaign
- Extracting Search Terms In The Past 14 Days
- Split HTTP Response

### Node Details

#### Get Only 'Enabled/Active' Campaigns
- **Type and role:** `n8n-nodes-base.filter`; keeps only campaigns with status `ENABLED`.
- **Configuration choices:**
  - Condition compares `$json.status` to `ENABLED`.
  - Strict type validation and case-sensitive matching enabled by node defaults shown.
- **Key expressions or variables used:** `={{ $json.status }}`
- **Input and output connections:** Input from **Fetch Google Campaigns**; output to **Filtering For A Specific Search Google Campaign**.
- **Version-specific requirements:** Type version `2.2`.
- **Edge cases / failures:**
  - If campaign status field is absent or formatted differently, records may be dropped.
  - Strict case sensitivity means values like `enabled` would fail.
- **Sub-workflow reference:** None.

#### Filtering For A Specific Search Google Campaign
- **Type and role:** `n8n-nodes-base.if`; filters campaigns by name.
- **Configuration choices:**
  - Requires campaign name to contain `add_your_campain_name`.
  - Also requires campaign name to contain `search`.
  - Both conditions are combined with `and`.
- **Key expressions or variables used:** `={{ $json.name }}`
- **Input and output connections:** Input from **Get Only 'Enabled/Active' Campaigns**; true output is connected to **Extracting Search Terms In The Past 14 Days**.
- **Version-specific requirements:** Type version `2.2`.
- **Edge cases / failures:**
  - Typo in placeholder (`campain`) is only in the string label, but the actual value still must be replaced manually.
  - If your campaign naming convention differs, nothing will pass.
  - Case-sensitive `contains` may exclude names like `Search` if not lowercase.
- **Sub-workflow reference:** None.

#### Extracting Search Terms In The Past 14 Days
- **Type and role:** `n8n-nodes-base.httpRequest`; performs a raw Google Ads Search API query.
- **Configuration choices:**
  - POST request to `https://googleads.googleapis.com/v19/customers/(add_your_google_ADS_ID)/googleAds:search`
  - Uses predefined Google Ads OAuth2 credentials.
  - Sends headers:
    - `developer-token: add_developer_token`
    - `login-customer-id: add_MCC_ID`
  - Sends body parameter `query` containing GAQL:
    - Selects search term, status, impressions, clicks, cost, conversions, campaign, ad group, and date.
    - Filters on `campaign.id = {{ $json.id }}`
    - Uses `segments.date DURING LAST_7_DAYS`
    - Requires `metrics.clicks > 1`
    - Limits to 500 rows.
- **Key expressions or variables used:**
  - `{{ $json.id }}` from the filtered campaign.
- **Input and output connections:** Input from **Filtering For A Specific Search Google Campaign**; output to **Split HTTP Response**.
- **Version-specific requirements:** Type version `4.2`.
- **Edge cases / failures:**
  - The node name says “Past 14 Days” but the actual query uses `LAST_7_DAYS`; this mismatch should be corrected if 14 days is intended.
  - Placeholder customer ID, developer token, and MCC ID must be replaced.
  - Google Ads API may reject requests for invalid login/customer relationships.
  - Large accounts may exceed the 500-row limit, causing partial analysis.
  - Query syntax errors or API version changes can break the call.
  - Search terms may be privacy-thresholded or missing depending on account volume and Google Ads constraints.
- **Sub-workflow reference:** None.

#### Split HTTP Response
- **Type and role:** `n8n-nodes-base.splitOut`; expands the `results` array from the HTTP response into individual items.
- **Configuration choices:** Splits out the `results` field.
- **Key expressions or variables used:** Field name `results`.
- **Input and output connections:** Input from **Extracting Search Terms In The Past 14 Days**; output to **Code (Google Ads Data Cleaner & Metrics Calculator)**.
- **Version-specific requirements:** Type version `1`.
- **Edge cases / failures:**
  - If the HTTP response does not contain `results`, downstream nodes receive nothing or fail.
  - API error payloads will not match the expected structure.
- **Sub-workflow reference:** None.

---

## 2.3 Data Cleaning, Filtering, and Aggregation

### Overview
This block transforms raw Google Ads search term rows into a flatter, easier-to-analyze dataset, removes terms already excluded or clearly branded, and aggregates repeated rows into consolidated search term metrics.

### Nodes Involved
- Code (Google Ads Data Cleaner & Metrics Calculator)
- Filter (Filtering Out Brand and Excluded Terms)
- Code (Google Ads Metrics Aggregator)
- Prepare Data for LLM
- Brand Details (Purpose: Adds context about your brand for AI analysis)

### Node Details

#### Code (Google Ads Data Cleaner & Metrics Calculator)
- **Type and role:** `n8n-nodes-base.code`; custom JavaScript transformation of Google Ads API response rows.
- **Configuration choices:**
  - Reads all input items using `$input.all()`.
  - Extracts:
    - campaign name
    - campaign ID parsed from `campaign.resourceName`
    - ad group name and ID
    - clicks, impressions, conversions, allConversions
    - `costMicros`, converted to `costUSD`
    - search term and status
    - date
  - Calculates:
    - CTR as percentage string
    - CPC as string with 2 decimals
- **Key expressions or variables used:** JavaScript access to `item.json`.
- **Input and output connections:** Input from **Split HTTP Response**; output to **Filter (Filtering Out Brand and Excluded Terms)**.
- **Version-specific requirements:** Type version `2`.
- **Edge cases / failures:**
  - If response keys use different casing than expected, fields may be empty.
  - `campaign.resourceName` parsing assumes a slash-delimited path.
  - `costUSD`, `ctr`, and `cpc` are partly string-formatted, which is fine for display but less ideal for later numeric analysis.
  - Missing nested objects are tolerated because the code checks for presence.
- **Sub-workflow reference:** None.

#### Filter (Filtering Out Brand and Excluded Terms)
- **Type and role:** `n8n-nodes-base.filter`; removes branded search terms and already excluded terms.
- **Configuration choices:**
  - Keeps items only if:
    - `$json.searchTerm` does **not** contain `add_your_brand_team`
    - `$json.status` is **not** `EXCLUDED`
- **Key expressions or variables used:**
  - `={{ $json.searchTerm }}`
  - `={{ $json.status }}`
- **Input and output connections:** Input from **Code (Google Ads Data Cleaner & Metrics Calculator)**; output to **Code (Google Ads Metrics Aggregator)**.
- **Version-specific requirements:** Type version `2.2`.
- **Edge cases / failures:**
  - Placeholder text appears to contain a typo: `brand_team` likely meant `brand_term`.
  - Single string exclusion is usually insufficient for real brand protection; multiple brand variants may be needed.
  - Case-sensitive matching may miss capitalization variants.
- **Sub-workflow reference:** None.

#### Code (Google Ads Metrics Aggregator)
- **Type and role:** `n8n-nodes-base.code`; aggregates multiple rows by search term + campaign + ad group.
- **Configuration choices:**
  - Builds a compound key:
    - `searchTerm_campaignId_adGroupId`
  - Sums:
    - clicks
    - impressions
    - costMicros
    - costUSD
    - conversions
    - allConversions
  - Preserves campaign/ad group/search term metadata.
- **Key expressions or variables used:** JavaScript object aggregation.
- **Input and output connections:** Input from **Filter (Filtering Out Brand and Excluded Terms)**; output to **Prepare Data for LLM**.
- **Version-specific requirements:** Type version `2`.
- **Edge cases / failures:**
  - The code uses `for (const item of items)` instead of `$input.all()`; in n8n Code node this often works because `items` is available, but behavior depends on runtime conventions. If execution fails, switch to `const items = $input.all();`.
  - Compound key can theoretically collide if underscores appear in IDs/terms in pathological ways, though unlikely.
  - Aggregated `costUSD` is summed from rounded row values, introducing minor rounding drift versus summing micros then converting once.
- **Sub-workflow reference:** None.

#### Prepare Data for LLM
- **Type and role:** `n8n-nodes-base.aggregate`; combines all processed items into a single item for AI consumption.
- **Configuration choices:** Uses `aggregateAllItemData`, producing one item with a `data` array containing all incoming records.
- **Key expressions or variables used:** None.
- **Input and output connections:** Input from **Code (Google Ads Metrics Aggregator)**; output to **Brand Details (Purpose: Adds context about your brand for AI analysis)**.
- **Version-specific requirements:** Type version `1`.
- **Edge cases / failures:**
  - If upstream returns no rows, the AI block may receive an empty dataset.
  - Large datasets may produce large prompts, increasing token cost or causing model context limits.
- **Sub-workflow reference:** None.

#### Brand Details (Purpose: Adds context about your brand for AI analysis)
- **Type and role:** `n8n-nodes-base.set`; adds brand context alongside the aggregated data.
- **Configuration choices:**
  - Sets `data` to the incoming aggregated `data`.
  - Sets `brand_info` to static placeholder text `add_your_brand_information`.
- **Key expressions or variables used:**
  - `={{ $json.data }}`
- **Input and output connections:** Input from **Prepare Data for LLM**; output to **AI Agent**.
- **Version-specific requirements:** Type version `3.4`.
- **Edge cases / failures:**
  - Placeholder must be replaced with meaningful brand guidance.
  - If `data` is missing, the AI will receive incomplete context.
- **Sub-workflow reference:** None.

---

## 2.4 AI Analysis Setup and Execution

### Overview
This block prepares an AI agent to classify non-brand search terms into wasted spend vs review-needed terms, using a selectable model backend and a structured output schema to keep the result machine-readable.

### Nodes Involved
- DeepSeek Chat Model
- OpenAI Chat Model
- xAI Grok Chat Model
- Anthropic Chat Model
- Model Selector (Chooses which AI model to use for analysis)
- Ensures AI output follows specific JSON format
- Get a campaign in Google Ads
- AI Agent

### Node Details

#### DeepSeek Chat Model
- **Type and role:** `@n8n/n8n-nodes-langchain.lmChatDeepSeek`; LLM backend option.
- **Configuration choices:** No special options configured.
- **Key expressions or variables used:** None.
- **Input and output connections:** Outputs AI language model connection to **Model Selector (Chooses which AI model to use for analysis)** input index 0.
- **Version-specific requirements:** Type version `1`.
- **Edge cases / failures:**
  - Requires valid DeepSeek API credentials.
  - Model availability and output quality may vary.
- **Sub-workflow reference:** None.

#### OpenAI Chat Model
- **Type and role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`; LLM backend option.
- **Configuration choices:**
  - Model set to `gpt-4.1-mini`.
  - No additional built-in tools enabled.
- **Key expressions or variables used:** None.
- **Input and output connections:** Outputs AI language model connection to **Model Selector (Chooses which AI model to use for analysis)** input index 1.
- **Version-specific requirements:** Type version `1.3`.
- **Edge cases / failures:**
  - Requires OpenAI credentials and model access.
  - Model naming can change over time.
- **Sub-workflow reference:** None.

#### xAI Grok Chat Model
- **Type and role:** `@n8n/n8n-nodes-langchain.lmChatXAiGrok`; LLM backend option.
- **Configuration choices:** No special options configured.
- **Key expressions or variables used:** None.
- **Input and output connections:** Outputs AI language model connection to **Model Selector (Chooses which AI model to use for analysis)** input index 2.
- **Version-specific requirements:** Type version `1`.
- **Edge cases / failures:**
  - Requires xAI credentials and supported environment.
- **Sub-workflow reference:** None.

#### Anthropic Chat Model
- **Type and role:** `@n8n/n8n-nodes-langchain.lmChatAnthropic`; LLM backend option.
- **Configuration choices:**
  - Model set to `claude-sonnet-4-5-20250929` with cached name “Claude Sonnet 4.5”.
- **Key expressions or variables used:** None.
- **Input and output connections:** Outputs AI language model connection to **Model Selector (Chooses which AI model to use for analysis)** input index 3.
- **Version-specific requirements:** Type version `1.3`.
- **Edge cases / failures:**
  - Requires Anthropic credentials and access to the configured model.
  - The specified future-dated model identifier may not exist in another environment.
- **Sub-workflow reference:** None.

#### Model Selector (Chooses which AI model to use for analysis)
- **Type and role:** `@n8n/n8n-nodes-langchain.modelSelector`; routes one of multiple LLM nodes into the AI Agent.
- **Configuration choices:**
  - `numberInputs` is `4`, matching the four connected model nodes.
  - Contains one rule with condition `0 equals 0`, which is always true.
  - In practice, this appears to force deterministic selection of the first matching path according to selector behavior, likely making this more of a placeholder than true model routing logic.
- **Key expressions or variables used:** Static numeric comparison.
- **Input and output connections:** Receives AI model inputs from the four model nodes; outputs selected model to **AI Agent**.
- **Version-specific requirements:** Type version `1`.
- **Edge cases / failures:**
  - The current rule does not implement meaningful routing logic.
  - Actual selected model depends on selector semantics in your n8n version.
  - If you need deterministic model choice, simplify by connecting one model directly or use a real conditional rule.
- **Sub-workflow reference:** None.

#### Ensures AI output follows specific JSON format
- **Type and role:** `@n8n/n8n-nodes-langchain.outputParserStructured`; enforces structured JSON output from the AI Agent.
- **Configuration choices:**
  - Provides a JSON schema example with fields:
    - `reportTitle`
    - `date`
    - `period`
    - `totalAdWastageUSD`
    - `summary`
    - `recommendations`
    - `keywordsForReview`
- **Key expressions or variables used:** None.
- **Input and output connections:** Connected to **AI Agent** as the output parser.
- **Version-specific requirements:** Type version `1.3`.
- **Edge cases / failures:**
  - If the model fails to comply with schema, the node or AI chain may error.
  - Schema example is not a full JSON Schema validator; behavior depends on node implementation.
- **Sub-workflow reference:** None.

#### Get a campaign in Google Ads
- **Type and role:** `n8n-nodes-base.googleAdsTool`; AI tool callable by the agent.
- **Configuration choices:**
  - Operation `get`.
  - `campaignId`, `clientCustomerId`, and `managerCustomerId` are supplied via `$fromAI(...)`.
  - This allows the agent to request additional campaign details if needed.
- **Key expressions or variables used:**
  - `$fromAI('Campaign_ID', '', 'string')`
  - `$fromAI('Client_Customer_ID', '', 'string')`
  - `$fromAI('Manager_Customer_ID', '', 'string')`
- **Input and output connections:** Connected to **AI Agent** as an AI tool.
- **Version-specific requirements:** Type version `1`.
- **Edge cases / failures:**
  - Tool use depends on the chosen model/tool-calling support.
  - If the model hallucinates IDs, tool calls may fail.
  - Credentials must match the same Google Ads environment as the main fetch.
- **Sub-workflow reference:** None.

#### AI Agent
- **Type and role:** `@n8n/n8n-nodes-langchain.agent`; core AI analysis node.
- **Configuration choices:**
  - Prompt input text is `={{ $json.data }}`.
  - Uses a detailed system message instructing the model to:
    - Act as a Google Ads SEM marketer
    - Use `brand_info` context
    - Identify non-brand terms
    - Separate terms into:
      - wastage = zero conversions
      - review = one or more conversions
    - Calculate `totalAdWastageUSD` from wastage only
    - Return strict JSON
  - Includes formatted current time via Dhaka timezone.
  - `hasOutputParser` is enabled.
- **Key expressions or variables used:**
  - `={{ $json.data }}`
  - `{{ $json.brand_info }}`
  - `{{ $now.setZone('Asia/Dhaka').toFormat('dd-MMM-yyyy HH:mm:ss') }}`
- **Input and output connections:**
  - Main input from **Brand Details (Purpose: Adds context about your brand for AI analysis)**
  - AI language model input from **Model Selector (Chooses which AI model to use for analysis)**
  - AI output parser from **Ensures AI output follows specific JSON format**
  - AI tool from **Get a campaign in Google Ads**
  - Main output to **Code (Converts AI analysis into formatted reports)**
- **Version-specific requirements:** Type version `3.1`.
- **Edge cases / failures:**
  - Large aggregated data arrays may exceed model context limits.
  - The model may still misclassify brand/non-brand terms if `brand_info` is weak.
  - If output parser rejects malformed output, execution stops.
  - Timezone formatting requires Luxon-compatible environment as used by n8n expressions.
- **Sub-workflow reference:** None.

---

## 2.5 Report Formatting and Distribution

### Overview
This block transforms the structured AI response into a Slack-style block payload and reuses that payload as message content across several communication platforms.

### Nodes Involved
- Code (Converts AI analysis into formatted reports)
- Send a message
- Create message
- Rapiwa
- Send a message1

### Node Details

#### Code (Converts AI analysis into formatted reports)
- **Type and role:** `n8n-nodes-base.code`; converts AI JSON output into a block-based report object.
- **Configuration choices:**
  - Reads `items[0].json.output`.
  - Builds a `blocks` array containing:
    - header
    - date/period fields
    - summary
    - total ad wastage
    - recommendations section
    - keywords for manual review
    - optional action buttons for approval/rejection of exclusions
  - Returns:
    - `blocksUi: { blocks }`
- **Key expressions or variables used:** JavaScript on `items[0].json.output`.
- **Input and output connections:** Input from **AI Agent**; output to all four notification nodes.
- **Version-specific requirements:** Type version `2`.
- **Edge cases / failures:**
  - Assumes `items[0].json.output` exists and matches schema.
  - Slack Block Kit structure is appropriate for Slack, but other channels may not render the same structure properly.
  - The action buttons are visual only here; no interaction handling workflow/webhook exists in this JSON.
- **Sub-workflow reference:** None.

#### Send a message
- **Type and role:** `n8n-nodes-base.slack`; sends the report to Slack.
- **Configuration choices:**
  - Message text/title: `Google Ads Search Term Analysis`
  - Sends to a specific channel ID placeholder `add_slack_channel_ID`
  - `messageType` is `block`
  - `blocksUi` is taken from `={{ $json.blocksUi }}`
  - Uses OAuth2 authentication.
- **Key expressions or variables used:** `={{ $json.blocksUi }}`
- **Input and output connections:** Input from **Code (Converts AI analysis into formatted reports)**; no downstream nodes.
- **Version-specific requirements:** Type version `2.3`.
- **Edge cases / failures:**
  - Invalid Slack channel ID or missing bot access to channel.
  - Block payload validation failures.
  - Interactive buttons require Slack app interactivity setup, not included here.
- **Sub-workflow reference:** None.

#### Create message
- **Type and role:** `n8n-nodes-base.microsoftTeams`; sends a message to a Teams channel.
- **Configuration choices:**
  - Resource: `channelMessage`
  - Team ID placeholder: `add_teams_ID`
  - Channel ID placeholder: `add_teams_channel_ID`
  - Message body: `={{ $json.blocksUi }}`
- **Key expressions or variables used:** `={{ $json.blocksUi }}`
- **Input and output connections:** Input from **Code (Converts AI analysis into formatted reports)**; no downstream nodes.
- **Version-specific requirements:** Type version `2`.
- **Edge cases / failures:**
  - Teams typically expects text/HTML, not Slack Block Kit JSON; this may send an object string or fail depending on node coercion.
  - OAuth permissions must allow posting to the selected team/channel.
- **Sub-workflow reference:** None.

#### Rapiwa
- **Type and role:** `n8n-nodes-rapiwa.rapiwa`; sends a WhatsApp-style message via Rapiwa.
- **Configuration choices:**
  - Number set to `880 1759-594891`
  - Message body: `={{ $json.blocksUi }}`
- **Key expressions or variables used:** `={{ $json.blocksUi }}`
- **Input and output connections:** Input from **Code (Converts AI analysis into formatted reports)**; no downstream nodes.
- **Version-specific requirements:** Community node type version `1`.
- **Edge cases / failures:**
  - Community node must be installed in the n8n instance.
  - Phone number and account must be valid in the provider.
  - Structured JSON block payload may not be appropriate message content for WhatsApp.
- **Sub-workflow reference:** None.

#### Send a message1
- **Type and role:** `n8n-nodes-base.discord`; posts the report to a Discord channel.
- **Configuration choices:**
  - Resource: `message`
  - `content` set to `={{ $json.blocksUi }}`
  - Guild/server ID placeholder: `add_server_ID`
  - Channel ID placeholder: `add_channel_ID`
- **Key expressions or variables used:** `={{ $json.blocksUi }}`
- **Input and output connections:** Input from **Code (Converts AI analysis into formatted reports)**; no downstream nodes.
- **Version-specific requirements:** Type version `2`.
- **Edge cases / failures:**
  - Discord content expects text, not Slack block JSON; formatting may be poor or invalid.
  - Bot must be present in the server/channel with send permissions.
- **Sub-workflow reference:** None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Schedule Trigger (Start Everyday at 8AM) | n8n-nodes-base.scheduleTrigger | Daily workflow trigger |  | Fetch Google Campaigns | ## Overview\nThis workflow automates Google Ads performance monitoring and optimization by analyzing search term data to identify wasteful spending and generate actionable recommendations. It uses AI to analyze campaign data, detect non-brand keywords that waste budget, and automatically generate daily reports with suggested negative keywords to exclude.\n\n## Workflow Steps\n* **Daily Trigger (8AM)**: Analyze previous day’s performance.\n* **Campaign & Search Term Data**: Fetch active campaigns and last 14 days of search terms; clean and aggregate metrics (CTR, CPC, conversions), excluding brand/irrelevant terms.\n* **AI Analysis**: Use DeepSeek, OpenAI (GPT-4.1-mini), xAI Grok, and Anthropic (Claude Sonnet 4.5) to identify wasteful keywords, suggest negatives, and calculate ad spend wastage.\n* **Report Generation & Distribution**: Build structured reports with recommendations and send to Teams, Discord, Slack, and Rapiwa.\n\n## Requirements:\nHere’s a concise version of your required resources:\n* **Google Ads API**: Access campaign and search term data.\n* **AI APIs**: DeepSeek, OpenAI (GPT-4.1-mini), xAI Grok, Anthropic (Claude Sonnet 4.5) for analysis.\n* **Communication Platforms**: Teams, Discord, Rapiwa (WhatsApp) credentials for notifications.\n* **Automation Platform**: n8n to orchestrate the workflow.\n\n## Customization\nHere’s a short version of your customization options:\n* **Analysis Period**: Change the search term timeframe (default 14 days).\n* **Brand Filtering**: Update excluded brand terms.\n* **AI Models**: Adjust model selection or routing logic.\n* **Report Content**: Edit prompts to add metrics, change criteria, or modify output format.\n* **Notifications**: Add/remove channels for report distribution.\n* **Campaign Targeting**: Switch which campaigns are monitored.\n\n## Useful Links\n* **Google Ads API**: [Documentation](https://developers.google.com/google-ads/api/docs/start)\n* **n8n Google Ads Node**: [Documentation](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.googleads/)\n# Trigger & Campaign Fetching Group |
| Fetch Google Campaigns | n8n-nodes-base.googleAds | Retrieve campaigns from Google Ads | Schedule Trigger (Start Everyday at 8AM) | Get Only 'Enabled/Active' Campaigns | ## Overview\nThis workflow automates Google Ads performance monitoring and optimization by analyzing search term data to identify wasteful spending and generate actionable recommendations. It uses AI to analyze campaign data, detect non-brand keywords that waste budget, and automatically generate daily reports with suggested negative keywords to exclude.\n\n## Workflow Steps\n* **Daily Trigger (8AM)**: Analyze previous day’s performance.\n* **Campaign & Search Term Data**: Fetch active campaigns and last 14 days of search terms; clean and aggregate metrics (CTR, CPC, conversions), excluding brand/irrelevant terms.\n* **AI Analysis**: Use DeepSeek, OpenAI (GPT-4.1-mini), xAI Grok, and Anthropic (Claude Sonnet 4.5) to identify wasteful keywords, suggest negatives, and calculate ad spend wastage.\n* **Report Generation & Distribution**: Build structured reports with recommendations and send to Teams, Discord, Slack, and Rapiwa.\n\n## Requirements:\nHere’s a concise version of your required resources:\n* **Google Ads API**: Access campaign and search term data.\n* **AI APIs**: DeepSeek, OpenAI (GPT-4.1-mini), xAI Grok, Anthropic (Claude Sonnet 4.5) for analysis.\n* **Communication Platforms**: Teams, Discord, Rapiwa (WhatsApp) credentials for notifications.\n* **Automation Platform**: n8n to orchestrate the workflow.\n\n## Customization\nHere’s a short version of your customization options:\n* **Analysis Period**: Change the search term timeframe (default 14 days).\n* **Brand Filtering**: Update excluded brand terms.\n* **AI Models**: Adjust model selection or routing logic.\n* **Report Content**: Edit prompts to add metrics, change criteria, or modify output format.\n* **Notifications**: Add/remove channels for report distribution.\n* **Campaign Targeting**: Switch which campaigns are monitored.\n\n## Useful Links\n* **Google Ads API**: [Documentation](https://developers.google.com/google-ads/api/docs/start)\n* **n8n Google Ads Node**: [Documentation](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.googleads/)\n# Trigger & Campaign Fetching Group |
| Get Only 'Enabled/Active' Campaigns | n8n-nodes-base.filter | Keep only enabled campaigns | Fetch Google Campaigns | Filtering For A Specific Search Google Campaign | ## Overview\nThis workflow automates Google Ads performance monitoring and optimization by analyzing search term data to identify wasteful spending and generate actionable recommendations. It uses AI to analyze campaign data, detect non-brand keywords that waste budget, and automatically generate daily reports with suggested negative keywords to exclude.\n\n## Workflow Steps\n* **Daily Trigger (8AM)**: Analyze previous day’s performance.\n* **Campaign & Search Term Data**: Fetch active campaigns and last 14 days of search terms; clean and aggregate metrics (CTR, CPC, conversions), excluding brand/irrelevant terms.\n* **AI Analysis**: Use DeepSeek, OpenAI (GPT-4.1-mini), xAI Grok, and Anthropic (Claude Sonnet 4.5) to identify wasteful keywords, suggest negatives, and calculate ad spend wastage.\n* **Report Generation & Distribution**: Build structured reports with recommendations and send to Teams, Discord, Slack, and Rapiwa.\n\n## Requirements:\nHere’s a concise version of your required resources:\n* **Google Ads API**: Access campaign and search term data.\n* **AI APIs**: DeepSeek, OpenAI (GPT-4.1-mini), xAI Grok, Anthropic (Claude Sonnet 4.5) for analysis.\n* **Communication Platforms**: Teams, Discord, Rapiwa (WhatsApp) credentials for notifications.\n* **Automation Platform**: n8n to orchestrate the workflow.\n\n## Customization\nHere’s a short version of your customization options:\n* **Analysis Period**: Change the search term timeframe (default 14 days).\n* **Brand Filtering**: Update excluded brand terms.\n* **AI Models**: Adjust model selection or routing logic.\n* **Report Content**: Edit prompts to add metrics, change criteria, or modify output format.\n* **Notifications**: Add/remove channels for report distribution.\n* **Campaign Targeting**: Switch which campaigns are monitored.\n\n## Useful Links\n* **Google Ads API**: [Documentation](https://developers.google.com/google-ads/api/docs/start)\n* **n8n Google Ads Node**: [Documentation](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.googleads/)\n## Campaign Filtering & Search Term Extraction Group |
| Filtering For A Specific Search Google Campaign | n8n-nodes-base.if | Keep only target search campaigns by name | Get Only 'Enabled/Active' Campaigns | Extracting Search Terms In The Past 14 Days | ## Overview\nThis workflow automates Google Ads performance monitoring and optimization by analyzing search term data to identify wasteful spending and generate actionable recommendations. It uses AI to analyze campaign data, detect non-brand keywords that waste budget, and automatically generate daily reports with suggested negative keywords to exclude.\n\n## Workflow Steps\n* **Daily Trigger (8AM)**: Analyze previous day’s performance.\n* **Campaign & Search Term Data**: Fetch active campaigns and last 14 days of search terms; clean and aggregate metrics (CTR, CPC, conversions), excluding brand/irrelevant terms.\n* **AI Analysis**: Use DeepSeek, OpenAI (GPT-4.1-mini), xAI Grok, and Anthropic (Claude Sonnet 4.5) to identify wasteful keywords, suggest negatives, and calculate ad spend wastage.\n* **Report Generation & Distribution**: Build structured reports with recommendations and send to Teams, Discord, Slack, and Rapiwa.\n\n## Requirements:\nHere’s a concise version of your required resources:\n* **Google Ads API**: Access campaign and search term data.\n* **AI APIs**: DeepSeek, OpenAI (GPT-4.1-mini), xAI Grok, Anthropic (Claude Sonnet 4.5) for analysis.\n* **Communication Platforms**: Teams, Discord, Rapiwa (WhatsApp) credentials for notifications.\n* **Automation Platform**: n8n to orchestrate the workflow.\n\n## Customization\nHere’s a short version of your customization options:\n* **Analysis Period**: Change the search term timeframe (default 14 days).\n* **Brand Filtering**: Update excluded brand terms.\n* **AI Models**: Adjust model selection or routing logic.\n* **Report Content**: Edit prompts to add metrics, change criteria, or modify output format.\n* **Notifications**: Add/remove channels for report distribution.\n* **Campaign Targeting**: Switch which campaigns are monitored.\n\n## Useful Links\n* **Google Ads API**: [Documentation](https://developers.google.com/google-ads/api/docs/start)\n* **n8n Google Ads Node**: [Documentation](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.googleads/)\n## Campaign Filtering & Search Term Extraction Group |
| Extracting Search Terms In The Past 14 Days | n8n-nodes-base.httpRequest | Query Google Ads search term performance via GAQL | Filtering For A Specific Search Google Campaign | Split HTTP Response | ## Overview\nThis workflow automates Google Ads performance monitoring and optimization by analyzing search term data to identify wasteful spending and generate actionable recommendations. It uses AI to analyze campaign data, detect non-brand keywords that waste budget, and automatically generate daily reports with suggested negative keywords to exclude.\n\n## Workflow Steps\n* **Daily Trigger (8AM)**: Analyze previous day’s performance.\n* **Campaign & Search Term Data**: Fetch active campaigns and last 14 days of search terms; clean and aggregate metrics (CTR, CPC, conversions), excluding brand/irrelevant terms.\n* **AI Analysis**: Use DeepSeek, OpenAI (GPT-4.1-mini), xAI Grok, and Anthropic (Claude Sonnet 4.5) to identify wasteful keywords, suggest negatives, and calculate ad spend wastage.\n* **Report Generation & Distribution**: Build structured reports with recommendations and send to Teams, Discord, Slack, and Rapiwa.\n\n## Requirements:\nHere’s a concise version of your required resources:\n* **Google Ads API**: Access campaign and search term data.\n* **AI APIs**: DeepSeek, OpenAI (GPT-4.1-mini), xAI Grok, Anthropic (Claude Sonnet 4.5) for analysis.\n* **Communication Platforms**: Teams, Discord, Rapiwa (WhatsApp) credentials for notifications.\n* **Automation Platform**: n8n to orchestrate the workflow.\n\n## Customization\nHere’s a short version of your customization options:\n* **Analysis Period**: Change the search term timeframe (default 14 days).\n* **Brand Filtering**: Update excluded brand terms.\n* **AI Models**: Adjust model selection or routing logic.\n* **Report Content**: Edit prompts to add metrics, change criteria, or modify output format.\n* **Notifications**: Add/remove channels for report distribution.\n* **Campaign Targeting**: Switch which campaigns are monitored.\n\n## Useful Links\n* **Google Ads API**: [Documentation](https://developers.google.com/google-ads/api/docs/start)\n* **n8n Google Ads Node**: [Documentation](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.googleads/)\n## Campaign Filtering & Search Term Extraction Group |
| Split HTTP Response | n8n-nodes-base.splitOut | Expand API result array into individual items | Extracting Search Terms In The Past 14 Days | Code (Google Ads Data Cleaner & Metrics Calculator) | ## Overview\nThis workflow automates Google Ads performance monitoring and optimization by analyzing search term data to identify wasteful spending and generate actionable recommendations. It uses AI to analyze campaign data, detect non-brand keywords that waste budget, and automatically generate daily reports with suggested negative keywords to exclude.\n\n## Workflow Steps\n* **Daily Trigger (8AM)**: Analyze previous day’s performance.\n* **Campaign & Search Term Data**: Fetch active campaigns and last 14 days of search terms; clean and aggregate metrics (CTR, CPC, conversions), excluding brand/irrelevant terms.\n* **AI Analysis**: Use DeepSeek, OpenAI (GPT-4.1-mini), xAI Grok, and Anthropic (Claude Sonnet 4.5) to identify wasteful keywords, suggest negatives, and calculate ad spend wastage.\n* **Report Generation & Distribution**: Build structured reports with recommendations and send to Teams, Discord, Slack, and Rapiwa.\n\n## Requirements:\nHere’s a concise version of your required resources:\n* **Google Ads API**: Access campaign and search term data.\n* **AI APIs**: DeepSeek, OpenAI (GPT-4.1-mini), xAI Grok, Anthropic (Claude Sonnet 4.5) for analysis.\n* **Communication Platforms**: Teams, Discord, Rapiwa (WhatsApp) credentials for notifications.\n* **Automation Platform**: n8n to orchestrate the workflow.\n\n## Customization\nHere’s a short version of your customization options:\n* **Analysis Period**: Change the search term timeframe (default 14 days).\n* **Brand Filtering**: Update excluded brand terms.\n* **AI Models**: Adjust model selection or routing logic.\n* **Report Content**: Edit prompts to add metrics, change criteria, or modify output format.\n* **Notifications**: Add/remove channels for report distribution.\n* **Campaign Targeting**: Switch which campaigns are monitored.\n\n## Useful Links\n* **Google Ads API**: [Documentation](https://developers.google.com/google-ads/api/docs/start)\n* **n8n Google Ads Node**: [Documentation](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.googleads/)\n## Campaign Filtering & Search Term Extraction Group |
| Code (Google Ads Data Cleaner & Metrics Calculator) | n8n-nodes-base.code | Flatten and normalize search term data; calculate CTR/CPC | Split HTTP Response | Filter (Filtering Out Brand and Excluded Terms) | ## Overview\nThis workflow automates Google Ads performance monitoring and optimization by analyzing search term data to identify wasteful spending and generate actionable recommendations. It uses AI to analyze campaign data, detect non-brand keywords that waste budget, and automatically generate daily reports with suggested negative keywords to exclude.\n\n## Workflow Steps\n* **Daily Trigger (8AM)**: Analyze previous day’s performance.\n* **Campaign & Search Term Data**: Fetch active campaigns and last 14 days of search terms; clean and aggregate metrics (CTR, CPC, conversions), excluding brand/irrelevant terms.\n* **AI Analysis**: Use DeepSeek, OpenAI (GPT-4.1-mini), xAI Grok, and Anthropic (Claude Sonnet 4.5) to identify wasteful keywords, suggest negatives, and calculate ad spend wastage.\n* **Report Generation & Distribution**: Build structured reports with recommendations and send to Teams, Discord, Slack, and Rapiwa.\n\n## Requirements:\nHere’s a concise version of your required resources:\n* **Google Ads API**: Access campaign and search term data.\n* **AI APIs**: DeepSeek, OpenAI (GPT-4.1-mini), xAI Grok, Anthropic (Claude Sonnet 4.5) for analysis.\n* **Communication Platforms**: Teams, Discord, Rapiwa (WhatsApp) credentials for notifications.\n* **Automation Platform**: n8n to orchestrate the workflow.\n\n## Customization\nHere’s a short version of your customization options:\n* **Analysis Period**: Change the search term timeframe (default 14 days).\n* **Brand Filtering**: Update excluded brand terms.\n* **AI Models**: Adjust model selection or routing logic.\n* **Report Content**: Edit prompts to add metrics, change criteria, or modify output format.\n* **Notifications**: Add/remove channels for report distribution.\n* **Campaign Targeting**: Switch which campaigns are monitored.\n\n## Useful Links\n* **Google Ads API**: [Documentation](https://developers.google.com/google-ads/api/docs/start)\n* **n8n Google Ads Node**: [Documentation](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.googleads/)\n## Data Processing & Cleaning Group |
| Filter (Filtering Out Brand and Excluded Terms) | n8n-nodes-base.filter | Remove branded and excluded search terms | Code (Google Ads Data Cleaner & Metrics Calculator) | Code (Google Ads Metrics Aggregator) | ## Overview\nThis workflow automates Google Ads performance monitoring and optimization by analyzing search term data to identify wasteful spending and generate actionable recommendations. It uses AI to analyze campaign data, detect non-brand keywords that waste budget, and automatically generate daily reports with suggested negative keywords to exclude.\n\n## Workflow Steps\n* **Daily Trigger (8AM)**: Analyze previous day’s performance.\n* **Campaign & Search Term Data**: Fetch active campaigns and last 14 days of search terms; clean and aggregate metrics (CTR, CPC, conversions), excluding brand/irrelevant terms.\n* **AI Analysis**: Use DeepSeek, OpenAI (GPT-4.1-mini), xAI Grok, and Anthropic (Claude Sonnet 4.5) to identify wasteful keywords, suggest negatives, and calculate ad spend wastage.\n* **Report Generation & Distribution**: Build structured reports with recommendations and send to Teams, Discord, Slack, and Rapiwa.\n\n## Requirements:\nHere’s a concise version of your required resources:\n* **Google Ads API**: Access campaign and search term data.\n* **AI APIs**: DeepSeek, OpenAI (GPT-4.1-mini), xAI Grok, Anthropic (Claude Sonnet 4.5) for analysis.\n* **Communication Platforms**: Teams, Discord, Rapiwa (WhatsApp) credentials for notifications.\n* **Automation Platform**: n8n to orchestrate the workflow.\n\n## Customization\nHere’s a short version of your customization options:\n* **Analysis Period**: Change the search term timeframe (default 14 days).\n* **Brand Filtering**: Update excluded brand terms.\n* **AI Models**: Adjust model selection or routing logic.\n* **Report Content**: Edit prompts to add metrics, change criteria, or modify output format.\n* **Notifications**: Add/remove channels for report distribution.\n* **Campaign Targeting**: Switch which campaigns are monitored.\n\n## Useful Links\n* **Google Ads API**: [Documentation](https://developers.google.com/google-ads/api/docs/start)\n* **n8n Google Ads Node**: [Documentation](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.googleads/)\n## Data Processing & Cleaning Group |
| Code (Google Ads Metrics Aggregator) | n8n-nodes-base.code | Aggregate metrics by search term, campaign, and ad group | Filter (Filtering Out Brand and Excluded Terms) | Prepare Data for LLM | ## Overview\nThis workflow automates Google Ads performance monitoring and optimization by analyzing search term data to identify wasteful spending and generate actionable recommendations. It uses AI to analyze campaign data, detect non-brand keywords that waste budget, and automatically generate daily reports with suggested negative keywords to exclude.\n\n## Workflow Steps\n* **Daily Trigger (8AM)**: Analyze previous day’s performance.\n* **Campaign & Search Term Data**: Fetch active campaigns and last 14 days of search terms; clean and aggregate metrics (CTR, CPC, conversions), excluding brand/irrelevant terms.\n* **AI Analysis**: Use DeepSeek, OpenAI (GPT-4.1-mini), xAI Grok, and Anthropic (Claude Sonnet 4.5) to identify wasteful keywords, suggest negatives, and calculate ad spend wastage.\n* **Report Generation & Distribution**: Build structured reports with recommendations and send to Teams, Discord, Slack, and Rapiwa.\n\n## Requirements:\nHere’s a concise version of your required resources:\n* **Google Ads API**: Access campaign and search term data.\n* **AI APIs**: DeepSeek, OpenAI (GPT-4.1-mini), xAI Grok, Anthropic (Claude Sonnet 4.5) for analysis.\n* **Communication Platforms**: Teams, Discord, Rapiwa (WhatsApp) credentials for notifications.\n* **Automation Platform**: n8n to orchestrate the workflow.\n\n## Customization\nHere’s a short version of your customization options:\n* **Analysis Period**: Change the search term timeframe (default 14 days).\n* **Brand Filtering**: Update excluded brand terms.\n* **AI Models**: Adjust model selection or routing logic.\n* **Report Content**: Edit prompts to add metrics, change criteria, or modify output format.\n* **Notifications**: Add/remove channels for report distribution.\n* **Campaign Targeting**: Switch which campaigns are monitored.\n\n## Useful Links\n* **Google Ads API**: [Documentation](https://developers.google.com/google-ads/api/docs/start)\n* **n8n Google Ads Node**: [Documentation](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.googleads/)\n## Data Processing & Cleaning Group |
| Prepare Data for LLM | n8n-nodes-base.aggregate | Combine all processed rows into one AI input payload | Code (Google Ads Metrics Aggregator) | Brand Details (Purpose: Adds context about your brand for AI analysis) | ## Overview\nThis workflow automates Google Ads performance monitoring and optimization by analyzing search term data to identify wasteful spending and generate actionable recommendations. It uses AI to analyze campaign data, detect non-brand keywords that waste budget, and automatically generate daily reports with suggested negative keywords to exclude.\n\n## Workflow Steps\n* **Daily Trigger (8AM)**: Analyze previous day’s performance.\n* **Campaign & Search Term Data**: Fetch active campaigns and last 14 days of search terms; clean and aggregate metrics (CTR, CPC, conversions), excluding brand/irrelevant terms.\n* **AI Analysis**: Use DeepSeek, OpenAI (GPT-4.1-mini), xAI Grok, and Anthropic (Claude Sonnet 4.5) to identify wasteful keywords, suggest negatives, and calculate ad spend wastage.\n* **Report Generation & Distribution**: Build structured reports with recommendations and send to Teams, Discord, Slack, and Rapiwa.\n\n## Requirements:\nHere’s a concise version of your required resources:\n* **Google Ads API**: Access campaign and search term data.\n* **AI APIs**: DeepSeek, OpenAI (GPT-4.1-mini), xAI Grok, Anthropic (Claude Sonnet 4.5) for analysis.\n* **Communication Platforms**: Teams, Discord, Rapiwa (WhatsApp) credentials for notifications.\n* **Automation Platform**: n8n to orchestrate the workflow.\n\n## Customization\nHere’s a short version of your customization options:\n* **Analysis Period**: Change the search term timeframe (default 14 days).\n* **Brand Filtering**: Update excluded brand terms.\n* **AI Models**: Adjust model selection or routing logic.\n* **Report Content**: Edit prompts to add metrics, change criteria, or modify output format.\n* **Notifications**: Add/remove channels for report distribution.\n* **Campaign Targeting**: Switch which campaigns are monitored.\n\n## Useful Links\n* **Google Ads API**: [Documentation](https://developers.google.com/google-ads/api/docs/start)\n* **n8n Google Ads Node**: [Documentation](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.googleads/)\n## Data Processing & Cleaning Group |
| Brand Details (Purpose: Adds context about your brand for AI analysis) | n8n-nodes-base.set | Add brand context for AI classification | Prepare Data for LLM | AI Agent | ## Overview\nThis workflow automates Google Ads performance monitoring and optimization by analyzing search term data to identify wasteful spending and generate actionable recommendations. It uses AI to analyze campaign data, detect non-brand keywords that waste budget, and automatically generate daily reports with suggested negative keywords to exclude.\n\n## Workflow Steps\n* **Daily Trigger (8AM)**: Analyze previous day’s performance.\n* **Campaign & Search Term Data**: Fetch active campaigns and last 14 days of search terms; clean and aggregate metrics (CTR, CPC, conversions), excluding brand/irrelevant terms.\n* **AI Analysis**: Use DeepSeek, OpenAI (GPT-4.1-mini), xAI Grok, and Anthropic (Claude Sonnet 4.5) to identify wasteful keywords, suggest negatives, and calculate ad spend wastage.\n* **Report Generation & Distribution**: Build structured reports with recommendations and send to Teams, Discord, Slack, and Rapiwa.\n\n## Requirements:\nHere’s a concise version of your required resources:\n* **Google Ads API**: Access campaign and search term data.\n* **AI APIs**: DeepSeek, OpenAI (GPT-4.1-mini), xAI Grok, Anthropic (Claude Sonnet 4.5) for analysis.\n* **Communication Platforms**: Teams, Discord, Rapiwa (WhatsApp) credentials for notifications.\n* **Automation Platform**: n8n to orchestrate the workflow.\n\n## Customization\nHere’s a short version of your customization options:\n* **Analysis Period**: Change the search term timeframe (default 14 days).\n* **Brand Filtering**: Update excluded brand terms.\n* **AI Models**: Adjust model selection or routing logic.\n* **Report Content**: Edit prompts to add metrics, change criteria, or modify output format.\n* **Notifications**: Add/remove channels for report distribution.\n* **Campaign Targeting**: Switch which campaigns are monitored.\n\n## Useful Links\n* **Google Ads API**: [Documentation](https://developers.google.com/google-ads/api/docs/start)\n* **n8n Google Ads Node**: [Documentation](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.googleads/)\n## AI Analysis Setup |
| DeepSeek Chat Model | @n8n/n8n-nodes-langchain.lmChatDeepSeek | AI model option |  | Model Selector (Chooses which AI model to use for analysis) | ## Overview\nThis workflow automates Google Ads performance monitoring and optimization by analyzing search term data to identify wasteful spending and generate actionable recommendations. It uses AI to analyze campaign data, detect non-brand keywords that waste budget, and automatically generate daily reports with suggested negative keywords to exclude.\n\n## Workflow Steps\n* **Daily Trigger (8AM)**: Analyze previous day’s performance.\n* **Campaign & Search Term Data**: Fetch active campaigns and last 14 days of search terms; clean and aggregate metrics (CTR, CPC, conversions), excluding brand/irrelevant terms.\n* **AI Analysis**: Use DeepSeek, OpenAI (GPT-4.1-mini), xAI Grok, and Anthropic (Claude Sonnet 4.5) to identify wasteful keywords, suggest negatives, and calculate ad spend wastage.\n* **Report Generation & Distribution**: Build structured reports with recommendations and send to Teams, Discord, Slack, and Rapiwa.\n\n## Requirements:\nHere’s a concise version of your required resources:\n* **Google Ads API**: Access campaign and search term data.\n* **AI APIs**: DeepSeek, OpenAI (GPT-4.1-mini), xAI Grok, Anthropic (Claude Sonnet 4.5) for analysis.\n* **Communication Platforms**: Teams, Discord, Rapiwa (WhatsApp) credentials for notifications.\n* **Automation Platform**: n8n to orchestrate the workflow.\n\n## Customization\nHere’s a short version of your customization options:\n* **Analysis Period**: Change the search term timeframe (default 14 days).\n* **Brand Filtering**: Update excluded brand terms.\n* **AI Models**: Adjust model selection or routing logic.\n* **Report Content**: Edit prompts to add metrics, change criteria, or modify output format.\n* **Notifications**: Add/remove channels for report distribution.\n* **Campaign Targeting**: Switch which campaigns are monitored.\n\n## Useful Links\n* **Google Ads API**: [Documentation](https://developers.google.com/google-ads/api/docs/start)\n* **n8n Google Ads Node**: [Documentation](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.googleads/)\n## AI Analysis Setup |
| OpenAI Chat Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | AI model option |  | Model Selector (Chooses which AI model to use for analysis) | ## Overview\nThis workflow automates Google Ads performance monitoring and optimization by analyzing search term data to identify wasteful spending and generate actionable recommendations. It uses AI to analyze campaign data, detect non-brand keywords that waste budget, and automatically generate daily reports with suggested negative keywords to exclude.\n\n## Workflow Steps\n* **Daily Trigger (8AM)**: Analyze previous day’s performance.\n* **Campaign & Search Term Data**: Fetch active campaigns and last 14 days of search terms; clean and aggregate metrics (CTR, CPC, conversions), excluding brand/irrelevant terms.\n* **AI Analysis**: Use DeepSeek, OpenAI (GPT-4.1-mini), xAI Grok, and Anthropic (Claude Sonnet 4.5) to identify wasteful keywords, suggest negatives, and calculate ad spend wastage.\n* **Report Generation & Distribution**: Build structured reports with recommendations and send to Teams, Discord, Slack, and Rapiwa.\n\n## Requirements:\nHere’s a concise version of your required resources:\n* **Google Ads API**: Access campaign and search term data.\n* **AI APIs**: DeepSeek, OpenAI (GPT-4.1-mini), xAI Grok, Anthropic (Claude Sonnet 4.5) for analysis.\n* **Communication Platforms**: Teams, Discord, Rapiwa (WhatsApp) credentials for notifications.\n* **Automation Platform**: n8n to orchestrate the workflow.\n\n## Customization\nHere’s a short version of your customization options:\n* **Analysis Period**: Change the search term timeframe (default 14 days).\n* **Brand Filtering**: Update excluded brand terms.\n* **AI Models**: Adjust model selection or routing logic.\n* **Report Content**: Edit prompts to add metrics, change criteria, or modify output format.\n* **Notifications**: Add/remove channels for report distribution.\n* **Campaign Targeting**: Switch which campaigns are monitored.\n\n## Useful Links\n* **Google Ads API**: [Documentation](https://developers.google.com/google-ads/api/docs/start)\n* **n8n Google Ads Node**: [Documentation](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.googleads/)\n## AI Analysis Setup |
| xAI Grok Chat Model | @n8n/n8n-nodes-langchain.lmChatXAiGrok | AI model option |  | Model Selector (Chooses which AI model to use for analysis) | ## Overview\nThis workflow automates Google Ads performance monitoring and optimization by analyzing search term data to identify wasteful spending and generate actionable recommendations. It uses AI to analyze campaign data, detect non-brand keywords that waste budget, and automatically generate daily reports with suggested negative keywords to exclude.\n\n## Workflow Steps\n* **Daily Trigger (8AM)**: Analyze previous day’s performance.\n* **Campaign & Search Term Data**: Fetch active campaigns and last 14 days of search terms; clean and aggregate metrics (CTR, CPC, conversions), excluding brand/irrelevant terms.\n* **AI Analysis**: Use DeepSeek, OpenAI (GPT-4.1-mini), xAI Grok, and Anthropic (Claude Sonnet 4.5) to identify wasteful keywords, suggest negatives, and calculate ad spend wastage.\n* **Report Generation & Distribution**: Build structured reports with recommendations and send to Teams, Discord, Slack, and Rapiwa.\n\n## Requirements:\nHere’s a concise version of your required resources:\n* **Google Ads API**: Access campaign and search term data.\n* **AI APIs**: DeepSeek, OpenAI (GPT-4.1-mini), xAI Grok, Anthropic (Claude Sonnet 4.5) for analysis.\n* **Communication Platforms**: Teams, Discord, Rapiwa (WhatsApp) credentials for notifications.\n* **Automation Platform**: n8n to orchestrate the workflow.\n\n## Customization\nHere’s a short version of your customization options:\n* **Analysis Period**: Change the search term timeframe (default 14 days).\n* **Brand Filtering**: Update excluded brand terms.\n* **AI Models**: Adjust model selection or routing logic.\n* **Report Content**: Edit prompts to add metrics, change criteria, or modify output format.\n* **Notifications**: Add/remove channels for report distribution.\n* **Campaign Targeting**: Switch which campaigns are monitored.\n\n## Useful Links\n* **Google Ads API**: [Documentation](https://developers.google.com/google-ads/api/docs/start)\n* **n8n Google Ads Node**: [Documentation](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.googleads/)\n## AI Analysis Setup |
| Anthropic Chat Model | @n8n/n8n-nodes-langchain.lmChatAnthropic | AI model option |  | Model Selector (Chooses which AI model to use for analysis) | ## Overview\nThis workflow automates Google Ads performance monitoring and optimization by analyzing search term data to identify wasteful spending and generate actionable recommendations. It uses AI to analyze campaign data, detect non-brand keywords that waste budget, and automatically generate daily reports with suggested negative keywords to exclude.\n\n## Workflow Steps\n* **Daily Trigger (8AM)**: Analyze previous day’s performance.\n* **Campaign & Search Term Data**: Fetch active campaigns and last 14 days of search terms; clean and aggregate metrics (CTR, CPC, conversions), excluding brand/irrelevant terms.\n* **AI Analysis**: Use DeepSeek, OpenAI (GPT-4.1-mini), xAI Grok, and Anthropic (Claude Sonnet 4.5) to identify wasteful keywords, suggest negatives, and calculate ad spend wastage.\n* **Report Generation & Distribution**: Build structured reports with recommendations and send to Teams, Discord, Slack, and Rapiwa.\n\n## Requirements:\nHere’s a concise version of your required resources:\n* **Google Ads API**: Access campaign and search term data.\n* **AI APIs**: DeepSeek, OpenAI (GPT-4.1-mini), xAI Grok, Anthropic (Claude Sonnet 4.5) for analysis.\n* **Communication Platforms**: Teams, Discord, Rapiwa (WhatsApp) credentials for notifications.\n* **Automation Platform**: n8n to orchestrate the workflow.\n\n## Customization\nHere’s a short version of your customization options:\n* **Analysis Period**: Change the search term timeframe (default 14 days).\n* **Brand Filtering**: Update excluded brand terms.\n* **AI Models**: Adjust model selection or routing logic.\n* **Report Content**: Edit prompts to add metrics, change criteria, or modify output format.\n* **Notifications**: Add/remove channels for report distribution.\n* **Campaign Targeting**: Switch which campaigns are monitored.\n\n## Useful Links\n* **Google Ads API**: [Documentation](https://developers.google.com/google-ads/api/docs/start)\n* **n8n Google Ads Node**: [Documentation](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.googleads/)\n## AI Analysis Setup |
| Model Selector (Chooses which AI model to use for analysis) | @n8n/n8n-nodes-langchain.modelSelector | Select or route among multiple AI models | DeepSeek Chat Model; OpenAI Chat Model; xAI Grok Chat Model; Anthropic Chat Model | AI Agent | ## Overview\nThis workflow automates Google Ads performance monitoring and optimization by analyzing search term data to identify wasteful spending and generate actionable recommendations. It uses AI to analyze campaign data, detect non-brand keywords that waste budget, and automatically generate daily reports with suggested negative keywords to exclude.\n\n## Workflow Steps\n* **Daily Trigger (8AM)**: Analyze previous day’s performance.\n* **Campaign & Search Term Data**: Fetch active campaigns and last 14 days of search terms; clean and aggregate metrics (CTR, CPC, conversions), excluding brand/irrelevant terms.\n* **AI Analysis**: Use DeepSeek, OpenAI (GPT-4.1-mini), xAI Grok, and Anthropic (Claude Sonnet 4.5) to identify wasteful keywords, suggest negatives, and calculate ad spend wastage.\n* **Report Generation & Distribution**: Build structured reports with recommendations and send to Teams, Discord, Slack, and Rapiwa.\n\n## Requirements:\nHere’s a concise version of your required resources:\n* **Google Ads API**: Access campaign and search term data.\n* **AI APIs**: DeepSeek, OpenAI (GPT-4.1-mini), xAI Grok, Anthropic (Claude Sonnet 4.5) for analysis.\n* **Communication Platforms**: Teams, Discord, Rapiwa (WhatsApp) credentials for notifications.\n* **Automation Platform**: n8n to orchestrate the workflow.\n\n## Customization\nHere’s a short version of your customization options:\n* **Analysis Period**: Change the search term timeframe (default 14 days).\n* **Brand Filtering**: Update excluded brand terms.\n* **AI Models**: Adjust model selection or routing logic.\n* **Report Content**: Edit prompts to add metrics, change criteria, or modify output format.\n* **Notifications**: Add/remove channels for report distribution.\n* **Campaign Targeting**: Switch which campaigns are monitored.\n\n## Useful Links\n* **Google Ads API**: [Documentation](https://developers.google.com/google-ads/api/docs/start)\n* **n8n Google Ads Node**: [Documentation](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.googleads/)\n## AI Analysis Setup |
| Ensures AI output follows specific JSON format | @n8n/n8n-nodes-langchain.outputParserStructured | Enforce structured AI output |  | AI Agent | ## Overview\nThis workflow automates Google Ads performance monitoring and optimization by analyzing search term data to identify wasteful spending and generate actionable recommendations. It uses AI to analyze campaign data, detect non-brand keywords that waste budget, and automatically generate daily reports with suggested negative keywords to exclude.\n\n## Workflow Steps\n* **Daily Trigger (8AM)**: Analyze previous day’s performance.\n* **Campaign & Search Term Data**: Fetch active campaigns and last 14 days of search terms; clean and aggregate metrics (CTR, CPC, conversions), excluding brand/irrelevant terms.\n* **AI Analysis**: Use DeepSeek, OpenAI (GPT-4.1-mini), xAI Grok, and Anthropic (Claude Sonnet 4.5) to identify wasteful keywords, suggest negatives, and calculate ad spend wastage.\n* **Report Generation & Distribution**: Build structured reports with recommendations and send to Teams, Discord, Slack, and Rapiwa.\n\n## Requirements:\nHere’s a concise version of your required resources:\n* **Google Ads API**: Access campaign and search term data.\n* **AI APIs**: DeepSeek, OpenAI (GPT-4.1-mini), xAI Grok, Anthropic (Claude Sonnet 4.5) for analysis.\n* **Communication Platforms**: Teams, Discord, Rapiwa (WhatsApp) credentials for notifications.\n* **Automation Platform**: n8n to orchestrate the workflow.\n\n## Customization\nHere’s a short version of your customization options:\n* **Analysis Period**: Change the search term timeframe (default 14 days).\n* **Brand Filtering**: Update excluded brand terms.\n* **AI Models**: Adjust model selection or routing logic.\n* **Report Content**: Edit prompts to add metrics, change criteria, or modify output format.\n* **Notifications**: Add/remove channels for report distribution.\n* **Campaign Targeting**: Switch which campaigns are monitored.\n\n## Useful Links\n* **Google Ads API**: [Documentation](https://developers.google.com/google-ads/api/docs/start)\n* **n8n Google Ads Node**: [Documentation](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.googleads/)\n## AI Analysis Setup |
| Get a campaign in Google Ads | n8n-nodes-base.googleAdsTool | Optional AI-callable tool for campaign lookup |  | AI Agent | ## Overview\nThis workflow automates Google Ads performance monitoring and optimization by analyzing search term data to identify wasteful spending and generate actionable recommendations. It uses AI to analyze campaign data, detect non-brand keywords that waste budget, and automatically generate daily reports with suggested negative keywords to exclude.\n\n## Workflow Steps\n* **Daily Trigger (8AM)**: Analyze previous day’s performance.\n* **Campaign & Search Term Data**: Fetch active campaigns and last 14 days of search terms; clean and aggregate metrics (CTR, CPC, conversions), excluding brand/irrelevant terms.\n* **AI Analysis**: Use DeepSeek, OpenAI (GPT-4.1-mini), xAI Grok, and Anthropic (Claude Sonnet 4.5) to identify wasteful keywords, suggest negatives, and calculate ad spend wastage.\n* **Report Generation & Distribution**: Build structured reports with recommendations and send to Teams, Discord, Slack, and Rapiwa.\n\n## Requirements:\nHere’s a concise version of your required resources:\n* **Google Ads API**: Access campaign and search term data.\n* **AI APIs**: DeepSeek, OpenAI (GPT-4.1-mini), xAI Grok, Anthropic (Claude Sonnet 4.5) for analysis.\n* **Communication Platforms**: Teams, Discord, Rapiwa (WhatsApp) credentials for notifications.\n* **Automation Platform**: n8n to orchestrate the workflow.\n\n## Customization\nHere’s a short version of your customization options:\n* **Analysis Period**: Change the search term timeframe (default 14 days).\n* **Brand Filtering**: Update excluded brand terms.\n* **AI Models**: Adjust model selection or routing logic.\n* **Report Content**: Edit prompts to add metrics, change criteria, or modify output format.\n* **Notifications**: Add/remove channels for report distribution.\n* **Campaign Targeting**: Switch which campaigns are monitored.\n\n## Useful Links\n* **Google Ads API**: [Documentation](https://developers.google.com/google-ads/api/docs/start)\n* **n8n Google Ads Node**: [Documentation](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.googleads/)\n## AI Analysis Setup |
| AI Agent | @n8n/n8n-nodes-langchain.agent | Analyze search terms and produce structured wastage report | Brand Details (Purpose: Adds context about your brand for AI analysis); Model Selector (Chooses which AI model to use for analysis); Ensures AI output follows specific JSON format); Get a campaign in Google Ads | Code (Converts AI analysis into formatted reports) | ## Overview\nThis workflow automates Google Ads performance monitoring and optimization by analyzing search term data to identify wasteful spending and generate actionable recommendations. It uses AI to analyze campaign data, detect non-brand keywords that waste budget, and automatically generate daily reports with suggested negative keywords to exclude.\n\n## Workflow Steps\n* **Daily Trigger (8AM)**: Analyze previous day’s performance.\n* **Campaign & Search Term Data**: Fetch active campaigns and last 14 days of search terms; clean and aggregate metrics (CTR, CPC, conversions), excluding brand/irrelevant terms.\n* **AI Analysis**: Use DeepSeek, OpenAI (GPT-4.1-mini), xAI Grok, and Anthropic (Claude Sonnet 4.5) to identify wasteful keywords, suggest negatives, and calculate ad spend wastage.\n* **Report Generation & Distribution**: Build structured reports with recommendations and send to Teams, Discord, Slack, and Rapiwa.\n\n## Requirements:\nHere’s a concise version of your required resources:\n* **Google Ads API**: Access campaign and search term data.\n* **AI APIs**: DeepSeek, OpenAI (GPT-4.1-mini), xAI Grok, Anthropic (Claude Sonnet 4.5) for analysis.\n* **Communication Platforms**: Teams, Discord, Rapiwa (WhatsApp) credentials for notifications.\n* **Automation Platform**: n8n to orchestrate the workflow.\n\n## Customization\nHere’s a short version of your customization options:\n* **Analysis Period**: Change the search term timeframe (default 14 days).\n* **Brand Filtering**: Update excluded brand terms.\n* **AI Models**: Adjust model selection or routing logic.\n* **Report Content**: Edit prompts to add metrics, change criteria, or modify output format.\n* **Notifications**: Add/remove channels for report distribution.\n* **Campaign Targeting**: Switch which campaigns are monitored.\n\n## Useful Links\n* **Google Ads API**: [Documentation](https://developers.google.com/google-ads/api/docs/start)\n* **n8n Google Ads Node**: [Documentation](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.googleads/)\n## AI Analysis Setup |
| Code (Converts AI analysis into formatted reports) | n8n-nodes-base.code | Build message blocks from AI output | AI Agent | Send a message; Create message; Rapiwa; Send a message1 | ## Overview\nThis workflow automates Google Ads performance monitoring and optimization by analyzing search term data to identify wasteful spending and generate actionable recommendations. It uses AI to analyze campaign data, detect non-brand keywords that waste budget, and automatically generate daily reports with suggested negative keywords to exclude.\n\n## Workflow Steps\n* **Daily Trigger (8AM)**: Analyze previous day’s performance.\n* **Campaign & Search Term Data**: Fetch active campaigns and last 14 days of search terms; clean and aggregate metrics (CTR, CPC, conversions), excluding brand/irrelevant terms.\n* **AI Analysis**: Use DeepSeek, OpenAI (GPT-4.1-mini), xAI Grok, and Anthropic (Claude Sonnet 4.5) to identify wasteful keywords, suggest negatives, and calculate ad spend wastage.\n* **Report Generation & Distribution**: Build structured reports with recommendations and send to Teams, Discord, Slack, and Rapiwa.\n\n## Requirements:\nHere’s a concise version of your required resources:\n* **Google Ads API**: Access campaign and search term data.\n* **AI APIs**: DeepSeek, OpenAI (GPT-4.1-mini), xAI Grok, Anthropic (Claude Sonnet 4.5) for analysis.\n* **Communication Platforms**: Teams, Discord, Rapiwa (WhatsApp) credentials for notifications.\n* **Automation Platform**: n8n to orchestrate the workflow.\n\n## Customization\nHere’s a short version of your customization options:\n* **Analysis Period**: Change the search term timeframe (default 14 days).\n* **Brand Filtering**: Update excluded brand terms.\n* **AI Models**: Adjust model selection or routing logic.\n* **Report Content**: Edit prompts to add metrics, change criteria, or modify output format.\n* **Notifications**: Add/remove channels for report distribution.\n* **Campaign Targeting**: Switch which campaigns are monitored.\n\n## Useful Links\n* **Google Ads API**: [Documentation](https://developers.google.com/google-ads/api/docs/start)\n* **n8n Google Ads Node**: [Documentation](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.googleads/)\n## Report Generation & Distribution Group |
| Send a message1 | n8n-nodes-base.discord | Send report to Discord | Code (Converts AI analysis into formatted reports) |  | ## Overview\nThis workflow automates Google Ads performance monitoring and optimization by analyzing search term data to identify wasteful spending and generate actionable recommendations. It uses AI to analyze campaign data, detect non-brand keywords that waste budget, and automatically generate daily reports with suggested negative keywords to exclude.\n\n## Workflow Steps\n* **Daily Trigger (8AM)**: Analyze previous day’s performance.\n* **Campaign & Search Term Data**: Fetch active campaigns and last 14 days of search terms; clean and aggregate metrics (CTR, CPC, conversions), excluding brand/irrelevant terms.\n* **AI Analysis**: Use DeepSeek, OpenAI (GPT-4.1-mini), xAI Grok, and Anthropic (Claude Sonnet 4.5) to identify wasteful keywords, suggest negatives, and calculate ad spend wastage.\n* **Report Generation & Distribution**: Build structured reports with recommendations and send to Teams, Discord, Slack, and Rapiwa.\n\n## Requirements:\nHere’s a concise version of your required resources:\n* **Google Ads API**: Access campaign and search term data.\n* **AI APIs**: DeepSeek, OpenAI (GPT-4.1-mini), xAI Grok, Anthropic (Claude Sonnet 4.5) for analysis.\n* **Communication Platforms**: Teams, Discord, Rapiwa (WhatsApp) credentials for notifications.\n* **Automation Platform**: n8n to orchestrate the workflow.\n\n## Customization\nHere’s a short version of your customization options:\n* **Analysis Period**: Change the search term timeframe (default 14 days).\n* **Brand Filtering**: Update excluded brand terms.\n* **AI Models**: Adjust model selection or routing logic.\n* **Report Content**: Edit prompts to add metrics, change criteria, or modify output format.\n* **Notifications**: Add/remove channels for report distribution.\n* **Campaign Targeting**: Switch which campaigns are monitored.\n\n## Useful Links\n* **Google Ads API**: [Documentation](https://developers.google.com/google-ads/api/docs/start)\n* **n8n Google Ads Node**: [Documentation](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.googleads/)\n## Report Generation & Distribution Group |
| Send a message | n8n-nodes-base.slack | Send report to Slack | Code (Converts AI analysis into formatted reports) |  | ## Overview\nThis workflow automates Google Ads performance monitoring and optimization by analyzing search term data to identify wasteful spending and generate actionable recommendations. It uses AI to analyze campaign data, detect non-brand keywords that waste budget, and automatically generate daily reports with suggested negative keywords to exclude.\n\n## Workflow Steps\n* **Daily Trigger (8AM)**: Analyze previous day’s performance.\n* **Campaign & Search Term Data**: Fetch active campaigns and last 14 days of search terms; clean and aggregate metrics (CTR, CPC, conversions), excluding brand/irrelevant terms.\n* **AI Analysis**: Use DeepSeek, OpenAI (GPT-4.1-mini), xAI Grok, and Anthropic (Claude Sonnet 4.5) to identify wasteful keywords, suggest negatives, and calculate ad spend wastage.\n* **Report Generation & Distribution**: Build structured reports with recommendations and send to Teams, Discord, Slack, and Rapiwa.\n\n## Requirements:\nHere’s a concise version of your required resources:\n* **Google Ads API**: Access campaign and search term data.\n* **AI APIs**: DeepSeek, OpenAI (GPT-4.1-mini), xAI Grok, Anthropic (Claude Sonnet 4.5) for analysis.\n* **Communication Platforms**: Teams, Discord, Rapiwa (WhatsApp) credentials for notifications.\n* **Automation Platform**: n8n to orchestrate the workflow.\n\n## Customization\nHere’s a short version of your customization options:\n* **Analysis Period**: Change the search term timeframe (default 14 days).\n* **Brand Filtering**: Update excluded brand terms.\n* **AI Models**: Adjust model selection or routing logic.\n* **Report Content**: Edit prompts to add metrics, change criteria, or modify output format.\n* **Notifications**: Add/remove channels for report distribution.\n* **Campaign Targeting**: Switch which campaigns are monitored.\n\n## Useful Links\n* **Google Ads API**: [Documentation](https://developers.google.com/google-ads/api/docs/start)\n* **n8n Google Ads Node**: [Documentation](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.googleads/)\n## Report Generation & Distribution Group |
| Create message | n8n-nodes-base.microsoftTeams | Send report to Microsoft Teams | Code (Converts AI analysis into formatted reports) |  | ## Overview\nThis workflow automates Google Ads performance monitoring and optimization by analyzing search term data to identify wasteful spending and generate actionable recommendations. It uses AI to analyze campaign data, detect non-brand keywords that waste budget, and automatically generate daily reports with suggested negative keywords to exclude.\n\n## Workflow Steps\n* **Daily Trigger (8AM)**: Analyze previous day’s performance.\n* **Campaign & Search Term Data**: Fetch active campaigns and last 14 days of search terms; clean and aggregate metrics (CTR, CPC, conversions), excluding brand/irrelevant terms.\n* **AI Analysis**: Use DeepSeek, OpenAI (GPT-4.1-mini), xAI Grok, and Anthropic (Claude Sonnet 4.5) to identify wasteful keywords, suggest negatives, and calculate ad spend wastage.\n* **Report Generation & Distribution**: Build structured reports with recommendations and send to Teams, Discord, Slack, and Rapiwa.\n\n## Requirements:\nHere’s a concise version of your required resources:\n* **Google Ads API**: Access campaign and search term data.\n* **AI APIs**: DeepSeek, OpenAI (GPT-4.1-mini), xAI Grok, Anthropic (Claude Sonnet 4.5) for analysis.\n* **Communication Platforms**: Teams, Discord, Rapiwa (WhatsApp) credentials for notifications.\n* **Automation Platform**: n8n to orchestrate the workflow.\n\n## Customization\nHere’s a short version of your customization options:\n* **Analysis Period**: Change the search term timeframe (default 14 days).\n* **Brand Filtering**: Update excluded brand terms.\n* **AI Models**: Adjust model selection or routing logic.\n* **Report Content**: Edit prompts to add metrics, change criteria, or modify output format.\n* **Notifications**: Add/remove channels for report distribution.\n* **Campaign Targeting**: Switch which campaigns are monitored.\n\n## Useful Links\n* **Google Ads API**: [Documentation](https://developers.google.com/google-ads/api/docs/start)\n* **n8n Google Ads Node**: [Documentation](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.googleads/)\n## Report Generation & Distribution Group |
| Rapiwa | n8n-nodes-rapiwa.rapiwa | Send report via Rapiwa/WhatsApp | Code (Converts AI analysis into formatted reports) |  | ## Overview\nThis workflow automates Google Ads performance monitoring and optimization by analyzing search term data to identify wasteful spending and generate actionable recommendations. It uses AI to analyze campaign data, detect non-brand keywords that waste budget, and automatically generate daily reports with suggested negative keywords to exclude.\n\n## Workflow Steps\n* **Daily Trigger (8AM)**: Analyze previous day’s performance.\n* **Campaign & Search Term Data**: Fetch active campaigns and last 14 days of search terms; clean and aggregate metrics (CTR, CPC, conversions), excluding brand/irrelevant terms.\n* **AI Analysis**: Use DeepSeek, OpenAI (GPT-4.1-mini), xAI Grok, and Anthropic (Claude Sonnet 4.5) to identify wasteful keywords, suggest negatives, and calculate ad spend wastage.\n* **Report Generation & Distribution**: Build structured reports with recommendations and send to Teams, Discord, Slack, and Rapiwa.\n\n## Requirements:\nHere’s a concise version of your required resources:\n* **Google Ads API**: Access campaign and search term data.\n* **AI APIs**: DeepSeek, OpenAI (GPT-4.1-mini), xAI Grok, Anthropic (Claude Sonnet 4.5) for analysis.\n* **Communication Platforms**: Teams, Discord, Rapiwa (WhatsApp) credentials for notifications.\n* **Automation Platform**: n8n to orchestrate the workflow.\n\n## Customization\nHere’s a short version of your customization options:\n* **Analysis Period**: Change the search term timeframe (default 14 days).\n* **Brand Filtering**: Update excluded brand terms.\n* **AI Models**: Adjust model selection or routing logic.\n* **Report Content**: Edit prompts to add metrics, change criteria, or modify output format.\n* **Notifications**: Add/remove channels for report distribution.\n* **Campaign Targeting**: Switch which campaigns are monitored.\n\n## Useful Links\n* **Google Ads API**: [Documentation](https://developers.google.com/google-ads/api/docs/start)\n* **n8n Google Ads Node**: [Documentation](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.googleads/)\n## Report Generation & Distribution Group |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n and name it something like:
   - `Automated Google Ads Search Term Analysis & Alerts`

2. **Add a Schedule Trigger node**
   - Node type: `Schedule Trigger`
   - Configure it to run once per day at `8:00`.
   - This becomes the primary entry point.

3. **Add a Google Ads node to fetch campaigns**
   - Node type: `Google Ads`
   - Connect it from the Schedule Trigger.
   - Configure Google Ads OAuth2 credentials.
   - Set:
     - `Client Customer ID` = your Google Ads client account ID
     - `Manager Customer ID` = your MCC account ID
   - Keep request options default unless needed.
   - If the node asks for a date range or additional options, set it as in the source workflow (`TODAY`).

4. **Add a Filter node for active campaigns**
   - Node type: `Filter`
   - Connect it after the Google Ads node.
   - Add one condition:
     - left value: `{{$json.status}}`
     - operator: `equals`
     - right value: `ENABLED`

5. **Add an If node for campaign name targeting**
   - Node type: `If`
   - Connect it after the active-campaign filter.
   - Add two conditions combined with `AND`:
     - `{{$json.name}}` contains your target campaign naming token, e.g. brand name
     - `{{$json.name}}` contains `search`
   - Use the `true` output for downstream processing.

6. **Add an HTTP Request node to query search terms**
   - Node type: `HTTP Request`
   - Connect it to the `true` output of the If node.
   - Configure:
     - Method: `POST`
     - URL: `https://googleads.googleapis.com/v19/customers/YOUR_GOOGLE_ADS_ID/googleAds:search`
     - Authentication: predefined credential type using Google Ads OAuth2
   - Add headers:
     - `developer-token`: your developer token
     - `login-customer-id`: your MCC ID
   - Enable request body.
   - Add body parameter `query`.
   - Use a GAQL query similar to:
     - select search term, status, metrics, campaign/ad group info, date
     - filter `campaign.id = {{ $json.id }}`
     - filter on date range such as `LAST_7_DAYS` or `LAST_14_DAYS`
     - filter `metrics.clicks > 1`
     - sort by clicks desc
     - limit 500
   - Important: decide whether you want 7 or 14 days and align the node name and prompt accordingly.

7. **Add a Split Out node**
   - Node type: `Split Out`
   - Connect it after the HTTP Request.
   - Set `Field To Split Out` = `results`

8. **Add a Code node to clean Google Ads response data**
   - Node type: `Code`
   - Connect it after `Split Out`.
   - Use JavaScript to:
     - read all input items
     - flatten nested campaign/ad group/search term fields
     - convert metrics to numbers
     - compute:
       - `costUSD = costMicros / 1,000,000`
       - CTR
       - CPC
   - Ensure output items contain at minimum:
     - `searchTerm`
     - `status`
     - `campaign`
     - `campaignId`
     - `adGroup`
     - `adGroupId`
     - `clicks`
     - `impressions`
     - `costMicros`
     - `costUSD`
     - `conversions`
     - `allConversions`
     - `date`

9. **Add a Filter node to remove brand and excluded terms**
   - Node type: `Filter`
   - Connect it after the cleaner code node.
   - Add conditions combined with `AND`:
     - `{{$json.searchTerm}}` does not contain your brand term
     - `{{$json.status}}` does not equal `EXCLUDED`
   - Replace placeholder brand text with real brand variants if possible.

10. **Add a Code node to aggregate metrics**
    - Node type: `Code`
    - Connect it after the brand/excluded filter.
    - Write JavaScript that groups by:
      - search term
      - campaign ID
      - ad group ID
    - Sum:
      - clicks
      - impressions
      - costMicros
      - costUSD
      - conversions
      - allConversions
    - Output one item per unique grouped combination.

11. **Add an Aggregate node**
    - Node type: `Aggregate`
    - Connect it after the aggregator code node.
    - Choose `Aggregate All Item Data`.
    - This creates one item with a `data` array for the AI prompt.

12. **Add a Set node for brand context**
    - Node type: `Set`
    - Connect it after the Aggregate node.
    - Create:
      - `data` = `{{$json.data}}`
      - `brand_info` = a descriptive string explaining your brand, products, naming variants, and what should be considered brand vs non-brand
    - This brand context is critical for AI accuracy.

13. **Add AI model nodes**
    Create these four nodes, even if you plan to use only one initially:
    - `DeepSeek Chat Model`
    - `OpenAI Chat Model`
    - `xAI Grok Chat Model`
    - `Anthropic Chat Model`

    Configure credentials for each service:
    - DeepSeek API credential
    - OpenAI API credential
    - xAI API credential
    - Anthropic API credential

    Suggested configuration from the workflow:
    - OpenAI model: `gpt-4.1-mini`
    - Anthropic model: `Claude Sonnet 4.5` equivalent available in your environment

14. **Add a Model Selector node**
    - Node type: `Model Selector`
    - Connect all four model nodes into it as AI language model inputs.
    - Set `numberInputs` to `4`.
    - If you want to match the source workflow exactly, create a rule that always evaluates true.
    - For a cleaner implementation, use just one model directly or build real routing logic based on cost/availability.

15. **Add a Structured Output Parser node**
    - Node type: `Structured Output Parser`
    - Define the expected JSON example with fields:
      - `reportTitle`
      - `date`
      - `period`
      - `totalAdWastageUSD`
      - `summary`
      - `recommendations`
      - `keywordsForReview`
    - Make sure arrays are always present, even if empty.

16. **Add a Google Ads Tool node**
    - Node type: `Google Ads Tool`
    - Configure operation `Get`.
    - Use AI-provided fields for:
      - campaign ID
      - client customer ID
      - manager customer ID
    - Connect this node to the AI Agent as an AI tool.
    - This is optional in practical terms, but it exists in the source workflow.

17. **Add the AI Agent node**
    - Node type: `AI Agent`
    - Connect the Set node into its main input.
    - Connect the Model Selector to its AI model input.
    - Connect the Structured Output Parser to its parser input.
    - Connect the Google Ads Tool node to its tool input.
    - Set the text input to:
      - `{{$json.data}}`
    - Add a system message instructing the model to:
      - act as a Google Ads SEM marketer
      - use `{{$json.brand_info}}`
      - identify non-brand search terms
      - classify zero-conversion terms as wastage
      - classify converting non-brand terms as review items
      - compute total ad wastage from zero-conversion terms only
      - return strict JSON in the exact schema
    - Optionally include current date/time using n8n expressions.

18. **Add a Code node to format the report**
    - Node type: `Code`
    - Connect it after the AI Agent.
    - Write JavaScript that reads the AI output and constructs a `blocksUi` object with Slack-style blocks:
      - header
      - date/period
      - summary
      - total ad wastage
      - recommendations
      - review keywords
      - action buttons
    - Return:
      - `{ json: { blocksUi: { blocks } } }`

19. **Add a Slack node**
    - Node type: `Slack`
    - Connect it after the report formatting code node.
    - Configure Slack OAuth2 credentials.
    - Set:
      - Channel ID = your Slack channel
      - Message type = `block`
      - Blocks UI = `{{$json.blocksUi}}`
      - Text fallback/title = `Google Ads Search Term Analysis`

20. **Add a Microsoft Teams node**
    - Node type: `Microsoft Teams`
    - Connect it after the report formatting code node.
    - Configure Teams OAuth2 credentials.
    - Set:
      - Resource = channel message
      - Team ID = your team
      - Channel ID = your channel
      - Message = `{{$json.blocksUi}}`
    - Recommended improvement: convert the report to plain text or HTML because Teams does not natively use Slack blocks.

21. **Add a Discord node**
    - Node type: `Discord`
    - Connect it after the report formatting code node.
    - Configure Discord bot credentials.
    - Set:
      - Guild/server ID
      - Channel ID
      - Content = `{{$json.blocksUi}}`
    - Recommended improvement: convert to plain text before sending.

22. **Add a Rapiwa node**
    - Node type: `Rapiwa`
    - Connect it after the report formatting code node.
    - Configure the Rapiwa credential.
    - Set:
      - Recipient number
      - Message = `{{$json.blocksUi}}`
    - Recommended improvement: send a text summary instead of raw block JSON.

23. **Review placeholders and replace all of them**
    Replace:
    - `add_your_client_ID`
    - `add_your_manager_ID`
    - `add_your_google_ADS_ID`
    - `add_developer_token`
    - `add_MCC_ID`
    - `add_your_campain_name`
    - `add_your_brand_team`
    - `add_your_brand_information`
    - channel/team/server IDs
    - phone number if needed

24. **Test each stage incrementally**
    - Run campaign fetch and verify output fields.
    - Run the search term API node and confirm `results` exists.
    - Inspect the cleaned and aggregated data.
    - Test the AI node with a small sample dataset first.
    - Validate the output parser behavior.
    - Confirm each messaging node can send successfully.

25. **Activate the workflow**
    - Once credentials and placeholders are configured and test executions are successful, activate the workflow so the schedule trigger runs automatically.

### Credential Setup Required

- **Google Ads OAuth2**
  - Used by:
    - Fetch Google Campaigns
    - Extracting Search Terms In The Past 14 Days
    - Get a campaign in Google Ads
- **DeepSeek API**
  - Used by DeepSeek Chat Model
- **OpenAI API**
  - Used by OpenAI Chat Model
- **xAI API**
  - Used by xAI Grok Chat Model
- **Anthropic API**
  - Used by Anthropic Chat Model
- **Slack OAuth2**
  - Used by Send a message
- **Microsoft Teams OAuth2**
  - Used by Create message
- **Discord Bot**
  - Used by Send a message1
- **Rapiwa API**
  - Used by Rapiwa

### Sub-workflow Setup
This workflow does **not** invoke any sub-workflow nodes. There are multiple service integrations, but no separate n8n child workflow.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| This workflow automates Google Ads performance monitoring and optimization by analyzing search term data to identify wasteful spending and generate actionable recommendations. It uses AI to analyze campaign data, detect non-brand keywords that waste budget, and automatically generate daily reports with suggested negative keywords to exclude. | Overall workflow purpose |
| Daily trigger at 8 AM; fetch campaigns and recent search terms; clean and aggregate metrics; analyze with AI; distribute reports to Teams, Discord, Slack, and Rapiwa. | Operational summary |
| Required resources include Google Ads API, AI APIs (DeepSeek, OpenAI, xAI, Anthropic), communication platform credentials, and n8n. | Deployment requirements |
| Customization options include analysis period, brand filtering logic, AI model selection, prompt/report content, notification channels, and campaign targeting rules. | Customization guidance |
| Google Ads API documentation | https://developers.google.com/google-ads/api/docs/start |
| n8n Google Ads node documentation | https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.googleads/ |

## Additional Implementation Notes
- The workflow title says “Analyze Google Ads search terms with AI and send wastage alerts,” and the internal workflow name is “Automated Google Ads Search Term Analysis & Alerts.”
- There is a mismatch between labeling and implementation:
  - node label says past 14 days
  - sticky note says last 14 days
  - actual GAQL query uses `LAST_7_DAYS`
- The Slack block payload is reused for Teams, Discord, and Rapiwa, but those services do not share Slack’s native rendering format. For production use, create channel-specific formatting branches.
- The Slack action buttons imply approval/rejection of negative keyword exclusions, but no follow-up webhook or action-handling branch exists in this workflow.