Track npm package downloads with Telegram commands and reports

https://n8nworkflows.xyz/workflows/track-npm-package-downloads-with-telegram-commands-and-reports-13945


# Track npm package downloads with Telegram commands and reports

# 1. Workflow Overview

This workflow monitors npm package downloads for a given maintainer and exposes the results through a Telegram bot plus scheduled Telegram reports. It supports interactive commands, weekly and monthly summary digests, and milestone alerts when cumulative downloads cross defined thresholds.

Typical use cases:
- Open-source maintainers tracking multiple npm packages
- Teams maintaining public SDKs or utility libraries
- Developers wanting lightweight package analytics through Telegram
- Automated periodic reporting without manually checking npm

## 1.1 Telegram Command Intake and Routing

This block receives Telegram messages, normalizes and fuzzy-matches command text, then routes execution to the correct reporting branch.

## 1.2 On-Demand Package Reports

This block handles the Telegram commands:
- `/downloads`
- `/weekly`
- `/status`
- `/trending`
- `/help`
- unknown commands

Each branch generates a Markdown message and replies to the originating Telegram chat.

## 1.3 Scheduled Weekly Digest

A cron trigger runs every Friday at 6 PM and sends a consolidated weekly npm performance digest to a fixed Telegram chat ID.

## 1.4 Scheduled Monthly Digest

A cron trigger runs on the 1st day of each month at 6 PM and sends a month-over-month summary plus all-time totals.

## 1.5 Daily Milestone Monitoring

A daily cron trigger checks whether any package crossed configured cumulative download milestones since the previous day. It sends a Telegram alert only if there are milestone crossings.

## 1.6 Documentation and Workspace Notes

Sticky notes document the workflow’s purpose, required edits, and an example output image.

---

# 2. Block-by-Block Analysis

## 2.1 Telegram Command Intake and Routing

### Overview
This block receives inbound Telegram messages, extracts the text command, performs typo-tolerant matching using Levenshtein distance, and routes the command to the proper branch. It is the interactive entry point of the workflow.

### Nodes Involved
- Telegram Trigger
- Parse Command
- Switch Command

### Node Details

#### Telegram Trigger
- **Type and technical role:** `n8n-nodes-base.telegramTrigger`; webhook-based Telegram bot entry node.
- **Configuration choices:** Listens for `message` updates only.
- **Key expressions or variables used:** None directly in configuration.
- **Input and output connections:** Entry point → outputs to `Parse Command`.
- **Version-specific requirements:** Type version `1.1`.
- **Edge cases or potential failure types:**
  - Telegram credential misconfiguration
  - Bot webhook registration issues
  - Bot not allowed in target chat/group
  - Non-text messages still arrive as `message` updates; downstream code assumes `message.text` may be absent and safely defaults to empty string
- **Sub-workflow reference:** None.

#### Parse Command
- **Type and technical role:** `n8n-nodes-base.code`; parses Telegram message text into a normalized command.
- **Configuration choices:** JavaScript code:
  - Reads message text from `$input.first().json?.message?.text || ''`
  - Reads chat ID from `$input.first().json?.message?.chat?.id`
  - Removes optional leading slash
  - Lowercases input
  - Compares input against supported commands using Levenshtein distance
  - Accepts typo correction when distance is `<= 4`
  - Emits `unknown` if no close match
- **Key expressions or variables used:**
  - `raw`
  - `chatId`
  - `normalized`
  - `commands = ['downloads', 'weekly', 'status', 'trending', 'help']`
  - `wasCorrected`
  - `correctedTo`
- **Input and output connections:** Input from `Telegram Trigger` → output to `Switch Command`.
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:**
  - Empty message text becomes `unknown`
  - Very short or unrelated strings may still fuzzy-match if distance threshold allows it
  - Commands with extra arguments are not explicitly supported; whole text is matched as one string
- **Sub-workflow reference:** None.

#### Switch Command
- **Type and technical role:** `n8n-nodes-base.switch`; routes execution by parsed command.
- **Configuration choices:** Six named outputs:
  - `downloads`
  - `weekly`
  - `status`
  - `trending`
  - `help`
  - `unknown`
  Each compares `={{ $json.command }}` with the expected literal.
- **Key expressions or variables used:** `{{$json.command}}`
- **Input and output connections:** Input from `Parse Command`; outputs to the corresponding fetch/message nodes.
- **Version-specific requirements:** Type version `3.4`; conditions use Switch rules version `3`.
- **Edge cases or potential failure types:**
  - If upstream payload lacks `command`, no rule would match
  - Strict type validation is enabled in conditions
- **Sub-workflow reference:** None.

---

## 2.2 On-Demand Package Reports

### Overview
This block generates Telegram replies for interactive commands. Most branches call npm APIs, build formatted Markdown summaries, and send the message back to the originating chat ID.

### Nodes Involved
- Fetch All-Time Downloads
- Send Downloads
- Fetch Weekly Downloads
- Send Weekly
- Fetch Status
- Send Status
- Fetch Trending
- Send Trending
- Help Message
- Send Help
- Unknown Command
- Send Unknown

### Node Details

#### Fetch All-Time Downloads
- **Type and technical role:** `n8n-nodes-base.code`; computes all-time download counts across discovered packages.
- **Configuration choices:**
  - Uses `NPM_USERNAME = 'monfortbrian'`
  - Attempts package auto-discovery via npm registry search:
    `https://registry.npmjs.org/-/v1/search?text=maintainer:${NPM_USERNAME}&size=50`
  - Falls back to a hardcoded package list if discovery fails
  - Queries npm downloads API for each package from `2020-01-01` to today
  - Sorts results descending by count
  - Builds Telegram Markdown output
  - Includes typo-correction note if `wasCorrected` is true
- **Key expressions or variables used:**
  - `$input.first().json.wasCorrected`
  - `$input.first().json.correctedTo`
  - `$input.first().json.chatId`
  - `grandTotal`
- **Input and output connections:** Input from `Switch Command` output `downloads` → output to `Send Downloads`.
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:**
  - npm registry search failure triggers fallback package list
  - npm downloads API failure per package yields `0` for that package
  - If `chatId` is missing, downstream Telegram send will fail
  - Start date is hardcoded to 2020-01-01, which may omit older package history
- **Sub-workflow reference:** None.

#### Send Downloads
- **Type and technical role:** `n8n-nodes-base.telegram`; sends all-time stats back to the requesting chat.
- **Configuration choices:**
  - `text = {{$json.message}}`
  - `chatId = {{$json.chatId}}`
  - `parse_mode = Markdown`
  - `appendAttribution = false`
- **Key expressions or variables used:** `{{$json.message}}`, `{{$json.chatId}}`
- **Input and output connections:** Input from `Fetch All-Time Downloads`; no downstream node.
- **Version-specific requirements:** Type version `1.2`.
- **Edge cases or potential failure types:**
  - Invalid Telegram credentials
  - Invalid `chatId`
  - Markdown parsing errors if package names or text contain special Markdown characters
  - Telegram message length limits if many packages exist
- **Sub-workflow reference:** None.

#### Fetch Weekly Downloads
- **Type and technical role:** `n8n-nodes-base.code`; retrieves last-week download counts per package.
- **Configuration choices:**
  - Same package discovery/fallback pattern as above
  - Uses `https://api.npmjs.org/downloads/point/last-week/${pkg}`
  - Sorts descending by weekly count
  - Adds a date range in display
- **Key expressions or variables used:**
  - `weeklyTotal`
  - `wasCorrected`
  - `correctedTo`
  - `chatId`
- **Input and output connections:** Input from `Switch Command` output `weekly` → output to `Send Weekly`.
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:**
  - Same npm API and message-length issues as above
  - JavaScript date arithmetic uses `new Date(now - 7 * 24 * 60 * 60 * 1000)`, which is valid but timezone-dependent in display
- **Sub-workflow reference:** None.

#### Send Weekly
- **Type and technical role:** `n8n-nodes-base.telegram`; sends weekly stats to the requesting chat.
- **Configuration choices:** Same as `Send Downloads`, but for weekly message content.
- **Key expressions or variables used:** `{{$json.message}}`, `{{$json.chatId}}`
- **Input and output connections:** Input from `Fetch Weekly Downloads`; no downstream node.
- **Version-specific requirements:** Type version `1.2`.
- **Edge cases or potential failure types:** Same Telegram/Markdown issues as other send nodes.
- **Sub-workflow reference:** None.

#### Fetch Status
- **Type and technical role:** `n8n-nodes-base.code`; combines weekly and all-time metrics in one response.
- **Configuration choices:**
  - Same package discovery/fallback logic
  - For each package fetches:
    - `last-week`
    - cumulative range `2020-01-01:today`
  - Builds two sorted sections:
    - weekly
    - all-time
- **Key expressions or variables used:**
  - `weeklyResults`
  - `allTimeResults`
  - `weeklyTotal`
  - `grandTotal`
  - `chatId`
- **Input and output connections:** Input from `Switch Command` output `status` → output to `Send Status`.
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:**
  - Same npm API fallback behavior
  - Per-package failures silently become zero values
  - Large result sets may exceed Telegram size limits
- **Sub-workflow reference:** None.

#### Send Status
- **Type and technical role:** `n8n-nodes-base.telegram`; sends combined weekly and all-time report.
- **Configuration choices:** Markdown mode, attribution disabled, dynamic chat ID.
- **Key expressions or variables used:** `{{$json.message}}`, `{{$json.chatId}}`
- **Input and output connections:** Input from `Fetch Status`; no downstream node.
- **Version-specific requirements:** Type version `1.2`.
- **Edge cases or potential failure types:** Same Telegram-related issues.
- **Sub-workflow reference:** None.

#### Fetch Trending
- **Type and technical role:** `n8n-nodes-base.code`; compares this week versus last week per package and calculates growth deltas and percentages.
- **Configuration choices:**
  - Same package discovery/fallback logic
  - Calculates date windows:
    - current week: `oneWeekAgo → now`
    - previous week: `twoWeeksAgo → oneWeekAgo`
  - Fetches both weekly windows for each package in parallel
  - Computes:
    - `delta`
    - `pct`
    - directional icon
  - Sorts by current week downloads
  - Highlights top package this week
- **Key expressions or variables used:**
  - `tw`, `lw`, `delta`, `pct`, `arrow`
  - `winner`
  - `chatId`
  - typo-correction fields
- **Input and output connections:** Input from `Switch Command` output `trending` → output to `Send Trending`.
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:**
  - `winner` assumes at least one package exists; empty package array would cause failure when accessing `winner.pkg`
  - If previous week downloads are zero and current week > 0, percentage is forced to `100`
  - API failures per package produce placeholder zero values with `🆕`
- **Sub-workflow reference:** None.

#### Send Trending
- **Type and technical role:** `n8n-nodes-base.telegram`; sends trending comparison to the requesting chat.
- **Configuration choices:** Markdown mode, attribution disabled, dynamic chat ID.
- **Key expressions or variables used:** `{{$json.message}}`, `{{$json.chatId}}`
- **Input and output connections:** Input from `Fetch Trending`; no downstream node.
- **Version-specific requirements:** Type version `1.2`.
- **Edge cases or potential failure types:** Same Telegram formatting and size issues.
- **Sub-workflow reference:** None.

#### Help Message
- **Type and technical role:** `n8n-nodes-base.code`; generates a static command help response.
- **Configuration choices:** Reads `chatId`, returns a list of supported commands and behavior notes.
- **Key expressions or variables used:** `$input.first().json.chatId`
- **Input and output connections:** Input from `Switch Command` output `help` → output to `Send Help`.
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:**
  - Missing `chatId` would break downstream send
- **Sub-workflow reference:** None.

#### Send Help
- **Type and technical role:** `n8n-nodes-base.telegram`; sends help text to the requesting chat.
- **Configuration choices:** Markdown mode, attribution disabled, dynamic chat ID.
- **Key expressions or variables used:** `{{$json.message}}`, `{{$json.chatId}}`
- **Input and output connections:** Input from `Help Message`; no downstream node.
- **Version-specific requirements:** Type version `1.2`.
- **Edge cases or potential failure types:** Same Telegram issues.
- **Sub-workflow reference:** None.

#### Unknown Command
- **Type and technical role:** `n8n-nodes-base.code`; builds an error/help response when command matching resolves to `unknown`.
- **Configuration choices:** Uses original text and returns the list of valid commands.
- **Key expressions or variables used:**
  - `$input.first().json.chatId`
  - `$input.first().json.original`
- **Input and output connections:** Input from `Switch Command` output `unknown` → output to `Send Unknown`.
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:**
  - Empty or missing original command will render as empty quoted string
- **Sub-workflow reference:** None.

#### Send Unknown
- **Type and technical role:** `n8n-nodes-base.telegram`; sends unrecognized-command response.
- **Configuration choices:** Markdown mode, attribution disabled, dynamic chat ID.
- **Key expressions or variables used:** `{{$json.message}}`, `{{$json.chatId}}`
- **Input and output connections:** Input from `Unknown Command`; no downstream node.
- **Version-specific requirements:** Type version `1.2`.
- **Edge cases or potential failure types:** Same Telegram issues.
- **Sub-workflow reference:** None.

---

## 2.3 Scheduled Weekly Digest

### Overview
This block runs automatically every Friday at 6 PM and sends a weekly performance digest to a fixed Telegram chat. It compares this week to the previous week and includes all-time totals.

### Nodes Involved
- Every Friday 6PM
- Fetch Weekly Digest
- Send Weekly Digest

### Node Details

#### Every Friday 6PM
- **Type and technical role:** `n8n-nodes-base.scheduleTrigger`; scheduled entry point.
- **Configuration choices:** Cron expression `0 18 * * 5` meaning Friday at 18:00.
- **Key expressions or variables used:** None.
- **Input and output connections:** Entry point → `Fetch Weekly Digest`.
- **Version-specific requirements:** Type version `1.2`.
- **Edge cases or potential failure types:**
  - Execution timezone depends on n8n instance/workflow timezone settings
  - Missed runs if instance is offline
- **Sub-workflow reference:** None.

#### Fetch Weekly Digest
- **Type and technical role:** `n8n-nodes-base.code`; builds the weekly digest message.
- **Configuration choices:**
  - Same package discovery/fallback logic
  - Calculates:
    - current week totals
    - previous week totals
    - all-time totals
  - Summarizes week-over-week delta
  - Highlights top package
  - Includes npm profile link: `https://www.npmjs.com/~monfortbrian`
  - Returns only `message`; no `chatId` because send node uses fixed chat ID
- **Key expressions or variables used:**
  - `weeklyTotal`
  - `prevWeekTotal`
  - `grandTotal`
  - `topPackage`
- **Input and output connections:** Input from `Every Friday 6PM` → output to `Send Weekly Digest`.
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:**
  - `topPackage` assumes non-empty package list
  - Fixed npm username in code and link must be edited together if customized
  - API failures for some packages become zeroed entries
- **Sub-workflow reference:** None.

#### Send Weekly Digest
- **Type and technical role:** `n8n-nodes-base.telegram`; posts weekly digest to a fixed Telegram chat.
- **Configuration choices:**
  - `text = {{$json.message}}`
  - fixed `chatId = "123456789"`
  - Markdown enabled
  - attribution disabled
- **Key expressions or variables used:** `{{$json.message}}`
- **Input and output connections:** Input from `Fetch Weekly Digest`; no downstream node.
- **Version-specific requirements:** Type version `1.2`.
- **Edge cases or potential failure types:**
  - Placeholder chat ID must be replaced
  - Bot must have permission to message that chat/user/channel
- **Sub-workflow reference:** None.

---

## 2.4 Scheduled Monthly Digest

### Overview
This block runs on the first day of each month at 6 PM and posts a month-over-month performance report plus cumulative totals.

### Nodes Involved
- 1st of Month 6PM
- Fetch Monthly Digest
- Send Monthly Digest

### Node Details

#### 1st of Month 6PM
- **Type and technical role:** `n8n-nodes-base.scheduleTrigger`; scheduled entry point.
- **Configuration choices:** Cron expression `0 18 1 * *` meaning day 1 of every month at 18:00.
- **Key expressions or variables used:** None.
- **Input and output connections:** Entry point → `Fetch Monthly Digest`.
- **Version-specific requirements:** Type version `1.2`.
- **Edge cases or potential failure types:** Same schedule/timezone concerns as other cron nodes.
- **Sub-workflow reference:** None.

#### Fetch Monthly Digest
- **Type and technical role:** `n8n-nodes-base.code`; computes month-over-month downloads and all-time totals.
- **Configuration choices:**
  - Same package discovery/fallback logic
  - Calculates:
    - last month range
    - previous month range
    - all-time range from `2020-01-01`
  - Computes month total, previous month total, grand total
  - Highlights top package of last month
  - Includes npm profile link
- **Key expressions or variables used:**
  - `lastMonthStart`, `lastMonthEnd`
  - `twoMonthsStart`, `twoMonthsEnd`
  - `monthTotal`, `prevMonthTotal`, `grandTotal`
  - `topPkg`
- **Input and output connections:** Input from `1st of Month 6PM` → output to `Send Monthly Digest`.
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:**
  - `topPkg` assumes at least one package exists
  - Date boundaries rely on server timezone and JavaScript Date behavior
  - Per-package API failures produce zero values
- **Sub-workflow reference:** None.

#### Send Monthly Digest
- **Type and technical role:** `n8n-nodes-base.telegram`; posts monthly digest to a fixed Telegram chat.
- **Configuration choices:**
  - dynamic text from `message`
  - fixed `chatId = "123456789"`
  - Markdown enabled
  - attribution disabled
- **Key expressions or variables used:** `{{$json.message}}`
- **Input and output connections:** Input from `Fetch Monthly Digest`; no downstream node.
- **Version-specific requirements:** Type version `1.2`.
- **Edge cases or potential failure types:**
  - Placeholder chat ID must be replaced
  - Telegram Markdown may fail if special characters are introduced
- **Sub-workflow reference:** None.

---

## 2.5 Daily Milestone Monitoring

### Overview
This block checks once per day whether any package has crossed a configured cumulative download milestone. It sends an alert only when milestone conditions are met, avoiding noise on uneventful days.

### Nodes Involved
- Milestone (daily check 9AM)
- Check Milestones
- Has Milestones?
- Send Milestone Alert

### Node Details

#### Milestone (daily check 9AM)
- **Type and technical role:** `n8n-nodes-base.scheduleTrigger`; daily cron entry point.
- **Configuration choices:** Cron expression `0 9 * * *` meaning every day at 09:00.
- **Key expressions or variables used:** None.
- **Input and output connections:** Entry point → `Check Milestones`.
- **Version-specific requirements:** Type version `1.2`.
- **Edge cases or potential failure types:** Same schedule/timezone concerns.
- **Sub-workflow reference:** None.

#### Check Milestones
- **Type and technical role:** `n8n-nodes-base.code`; determines whether packages crossed any thresholds between yesterday and today.
- **Configuration choices:**
  - Uses `NPM_USERNAME = 'monfortbrian'`
  - Uses milestone array:
    `[100, 500, 1000, 5000, 10000, 50000, 100000]`
  - Same package discovery/fallback logic
  - For each package queries cumulative all-time downloads up to:
    - today
    - yesterday
  - If `previous < milestone && current >= milestone`, records an alert
  - Returns:
    - `{ hasAlerts: false, message: null }` when none
    - `{ hasAlerts: true, message }` when one or more milestones crossed
- **Key expressions or variables used:**
  - `MILESTONES`
  - `today`
  - `yesterday`
  - `alerts`
- **Input and output connections:** Input from `Milestone (daily check 9AM)` → output to `Has Milestones?`.
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:**
  - Package API failures are swallowed silently
  - Multiple milestones can trigger for the same package in one day if growth is large
  - If API data is delayed, milestone notifications may arrive late
- **Sub-workflow reference:** None.

#### Has Milestones?
- **Type and technical role:** `n8n-nodes-base.if`; allows message delivery only when alerts exist.
- **Configuration choices:** Boolean condition `{{$json.hasAlerts}} == true`.
- **Key expressions or variables used:** `{{$json.hasAlerts}}`
- **Input and output connections:** Input from `Check Milestones`; true output goes to `Send Milestone Alert`. False branch is unused.
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:**
  - Missing `hasAlerts` field would cause condition mismatch and no send
- **Sub-workflow reference:** None.

#### Send Milestone Alert
- **Type and technical role:** `n8n-nodes-base.telegram`; sends milestone alert to a fixed Telegram chat.
- **Configuration choices:**
  - `text = {{$json.message}}`
  - fixed `chatId = "123456789"`
  - Markdown enabled
  - attribution disabled
- **Key expressions or variables used:** `{{$json.message}}`
- **Input and output connections:** Input from `Has Milestones?` true branch; no downstream node.
- **Version-specific requirements:** Type version `1.2`.
- **Edge cases or potential failure types:**
  - Placeholder chat ID must be replaced
  - Bot permissions required in destination chat
- **Sub-workflow reference:** None.

---

## 2.6 Documentation and Workspace Notes

### Overview
These nodes are visual documentation aids for users working inside the n8n canvas. They do not execute as part of the workflow logic.

### Nodes Involved
- Sticky Note
- Sticky Note9
- Sticky Note10
- Sticky Note11

### Node Details

#### Sticky Note9
- **Type and technical role:** `n8n-nodes-base.stickyNote`; top-level workflow description and setup guidance.
- **Configuration choices:** Large descriptive note explaining purpose, commands, reports, defaults, and setup steps.
- **Key expressions or variables used:** None.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:** None; non-executable.
- **Sub-workflow reference:** None.

#### Sticky Note10
- **Type and technical role:** `n8n-nodes-base.stickyNote`; highlights customization points for the command/report code block area.
- **Configuration choices:** Notes that `NPM_USERNAME` and Telegram `chatId` must be edited.
- **Key expressions or variables used:** None.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None.

#### Sticky Note11
- **Type and technical role:** `n8n-nodes-base.stickyNote`; highlights customization points for scheduled reporting and milestone monitoring.
- **Configuration choices:** Notes that cron schedules, `NPM_USERNAME`, and milestone thresholds can be customized.
- **Key expressions or variables used:** None.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None.

#### Sticky Note
- **Type and technical role:** `n8n-nodes-base.stickyNote`; displays an example output image.
- **Configuration choices:** Contains image link:
  `https://res.cloudinary.com/dewqzljou/image/upload/v1773000261/NPM_package_tracker_-_demo_a59te9.jpg`
- **Key expressions or variables used:** None.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:** External image availability only.
- **Sub-workflow reference:** None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Telegram Trigger | Telegram Trigger | Receives Telegram bot messages |  | Parse Command | # NPM package tracker<br>Monitor npm package downloads from Telegram with commands, weekly digests, and milestone alerts<br><br>## Description<br><br>This template is designed for developers and teams who publish packages to npm and want simple visibility into package adoption.<br><br>It is particularly useful for:<br><br>- Open-source maintainers managing multiple npm packages<br>- Developer tool creators shipping libraries, plugins, or integrations<br>- Engineering teams maintaining public SDKs or utilities<br>- Agencies or startups distributing reusable packages<br><br>Instead of manually checking npm statistics, this workflow delivers download insights automatically through Telegram.<br><br>## What it does<br><br>The workflow combines Telegram commands with scheduled reports to provide quick insights into package usage.<br><br>**Commands**<br><br>- `/downloads` - all-time totals, sorted highest first<br>- `/weekly` - last 7 days per package<br>- `/status` - weekly and all-time combined<br>- `/trending` - this week vs last week with delta and growth percentage<br>- `/help` - available commands<br><br>**Automated reports**<br><br>- Weekly digest with package performance<br>- Monthly summary comparing usage trends<br>- Daily milestone checker that sends alerts when packages cross download thresholds<br>The workflow automatically discovers packages published under your npm username, so new packages appear in reports without any changes.<br><br>**Smart defaults:**<br><br>- Typo-tolerant command matching via Levenshtein distance<br>- Slash optional, case insensitive<br>- Auto-discovers packages from npm registry - new packages appear without code changes<br>- Milestone check is silent on days with no crossings - zero noise<br><br>## How it works<br><br>1. A Telegram Trigger receives messages from the bot.<br>2. A command parser identifies which command was sent.<br>3. The workflow queries the npm Downloads API for package statistics.<br>4. Results are formatted and sent back to Telegram.<br>5. Scheduled triggers generate weekly and monthly reports and check milestone thresholds.<br><br>## Set up steps<br><br>Setup usually takes about **5–10 minutes**.<br><br>1. Run n8n (Cloud or self-hosted)<br>2. Create a Telegram bot via BotFather<br>3. Import the workflow into n8n<br>4. Add your Telegram bot credential<br>5. Set your npm username in the Code nodes<br>6. Activate the workflow |
| Parse Command | Code | Normalizes and fuzzy-matches Telegram commands | Telegram Trigger | Switch Command | # NPM package tracker<br>Monitor npm package downloads from Telegram with commands, weekly digests, and milestone alerts<br><br>## Description<br><br>This template is designed for developers and teams who publish packages to npm and want simple visibility into package adoption.<br><br>It is particularly useful for:<br><br>- Open-source maintainers managing multiple npm packages<br>- Developer tool creators shipping libraries, plugins, or integrations<br>- Engineering teams maintaining public SDKs or utilities<br>- Agencies or startups distributing reusable packages<br><br>Instead of manually checking npm statistics, this workflow delivers download insights automatically through Telegram.<br><br>## What it does<br><br>The workflow combines Telegram commands with scheduled reports to provide quick insights into package usage.<br><br>**Commands**<br><br>- `/downloads` - all-time totals, sorted highest first<br>- `/weekly` - last 7 days per package<br>- `/status` - weekly and all-time combined<br>- `/trending` - this week vs last week with delta and growth percentage<br>- `/help` - available commands<br><br>**Automated reports**<br><br>- Weekly digest with package performance<br>- Monthly summary comparing usage trends<br>- Daily milestone checker that sends alerts when packages cross download thresholds<br>The workflow automatically discovers packages published under your npm username, so new packages appear in reports without any changes.<br><br>**Smart defaults:**<br><br>- Typo-tolerant command matching via Levenshtein distance<br>- Slash optional, case insensitive<br>- Auto-discovers packages from npm registry - new packages appear without code changes<br>- Milestone check is silent on days with no crossings - zero noise<br><br>## How it works<br><br>1. A Telegram Trigger receives messages from the bot.<br>2. A command parser identifies which command was sent.<br>3. The workflow queries the npm Downloads API for package statistics.<br>4. Results are formatted and sent back to Telegram.<br>5. Scheduled triggers generate weekly and monthly reports and check milestone thresholds.<br><br>## Set up steps<br><br>Setup usually takes about **5–10 minutes**.<br><br>1. Run n8n (Cloud or self-hosted)<br>2. Create a Telegram bot via BotFather<br>3. Import the workflow into n8n<br>4. Add your Telegram bot credential<br>5. Set your npm username in the Code nodes<br>6. Activate the workflow |
| Switch Command | Switch | Routes parsed commands to branch-specific handlers | Parse Command | Fetch All-Time Downloads; Fetch Weekly Downloads; Fetch Status; Fetch Trending; Help Message; Unknown Command | # NPM package tracker<br>Monitor npm package downloads from Telegram with commands, weekly digests, and milestone alerts<br><br>## Description<br><br>This template is designed for developers and teams who publish packages to npm and want simple visibility into package adoption.<br><br>It is particularly useful for:<br><br>- Open-source maintainers managing multiple npm packages<br>- Developer tool creators shipping libraries, plugins, or integrations<br>- Engineering teams maintaining public SDKs or utilities<br>- Agencies or startups distributing reusable packages<br><br>Instead of manually checking npm statistics, this workflow delivers download insights automatically through Telegram.<br><br>## What it does<br><br>The workflow combines Telegram commands with scheduled reports to provide quick insights into package usage.<br><br>**Commands**<br><br>- `/downloads` - all-time totals, sorted highest first<br>- `/weekly` - last 7 days per package<br>- `/status` - weekly and all-time combined<br>- `/trending` - this week vs last week with delta and growth percentage<br>- `/help` - available commands<br><br>**Automated reports**<br><br>- Weekly digest with package performance<br>- Monthly summary comparing usage trends<br>- Daily milestone checker that sends alerts when packages cross download thresholds<br>The workflow automatically discovers packages published under your npm username, so new packages appear in reports without any changes.<br><br>**Smart defaults:**<br><br>- Typo-tolerant command matching via Levenshtein distance<br>- Slash optional, case insensitive<br>- Auto-discovers packages from npm registry - new packages appear without code changes<br>- Milestone check is silent on days with no crossings - zero noise<br><br>## How it works<br><br>1. A Telegram Trigger receives messages from the bot.<br>2. A command parser identifies which command was sent.<br>3. The workflow queries the npm Downloads API for package statistics.<br>4. Results are formatted and sent back to Telegram.<br>5. Scheduled triggers generate weekly and monthly reports and check milestone thresholds.<br><br>## Set up steps<br><br>Setup usually takes about **5–10 minutes**.<br><br>1. Run n8n (Cloud or self-hosted)<br>2. Create a Telegram bot via BotFather<br>3. Import the workflow into n8n<br>4. Add your Telegram bot credential<br>5. Set your npm username in the Code nodes<br>6. Activate the workflow |
| Fetch All-Time Downloads | Code | Builds all-time per-package download report | Switch Command | Send Downloads | ## What to change<br><br>1. NPM_USERNAME in the Code nodes - set your npm username<br><br>2. Telegram chatId in the digest and milestone nodes |
| Send Downloads | Telegram | Sends all-time report to requesting chat | Fetch All-Time Downloads |  | ## What to change<br><br>1. NPM_USERNAME in the Code nodes - set your npm username<br><br>2. Telegram chatId in the digest and milestone nodes |
| Fetch Weekly Downloads | Code | Builds last-week per-package download report | Switch Command | Send Weekly | ## What to change<br><br>1. NPM_USERNAME in the Code nodes - set your npm username<br><br>2. Telegram chatId in the digest and milestone nodes |
| Send Weekly | Telegram | Sends weekly report to requesting chat | Fetch Weekly Downloads |  | ## What to change<br><br>1. NPM_USERNAME in the Code nodes - set your npm username<br><br>2. Telegram chatId in the digest and milestone nodes |
| Fetch Status | Code | Builds combined weekly and all-time package status | Switch Command | Send Status | ## What to change<br><br>1. NPM_USERNAME in the Code nodes - set your npm username<br><br>2. Telegram chatId in the digest and milestone nodes |
| Send Status | Telegram | Sends combined status report to requesting chat | Fetch Status |  | ## What to change<br><br>1. NPM_USERNAME in the Code nodes - set your npm username<br><br>2. Telegram chatId in the digest and milestone nodes |
| Fetch Trending | Code | Builds week-vs-last-week trending report | Switch Command | Send Trending | ## What to change<br><br>1. NPM_USERNAME in the Code nodes - set your npm username<br><br>2. Telegram chatId in the digest and milestone nodes |
| Send Trending | Telegram | Sends trending report to requesting chat | Fetch Trending |  | ## What to change<br><br>1. NPM_USERNAME in the Code nodes - set your npm username<br><br>2. Telegram chatId in the digest and milestone nodes |
| Help Message | Code | Builds static help response | Switch Command | Send Help | ## What to change<br><br>1. NPM_USERNAME in the Code nodes - set your npm username<br><br>2. Telegram chatId in the digest and milestone nodes |
| Send Help | Telegram | Sends help message to requesting chat | Help Message |  | ## What to change<br><br>1. NPM_USERNAME in the Code nodes - set your npm username<br><br>2. Telegram chatId in the digest and milestone nodes |
| Unknown Command | Code | Builds fallback response for unsupported command | Switch Command | Send Unknown | ## What to change<br><br>1. NPM_USERNAME in the Code nodes - set your npm username<br><br>2. Telegram chatId in the digest and milestone nodes |
| Send Unknown | Telegram | Sends unknown-command response | Unknown Command |  | ## What to change<br><br>1. NPM_USERNAME in the Code nodes - set your npm username<br><br>2. Telegram chatId in the digest and milestone nodes |
| Every Friday 6PM | Schedule Trigger | Starts weekly digest on cron |  | Fetch Weekly Digest | ## What to change<br><br>1. Cron schedules if you want different reporting times<br><br>2. NPM_USERNAME in the Code nodes - set your npm username<br><br>3. MILESTONES array to customize alert thresholds |
| Fetch Weekly Digest | Code | Builds automatic weekly digest | Every Friday 6PM | Send Weekly Digest | ## What to change<br><br>1. Cron schedules if you want different reporting times<br><br>2. NPM_USERNAME in the Code nodes - set your npm username<br><br>3. MILESTONES array to customize alert thresholds |
| Send Weekly Digest | Telegram | Sends weekly digest to fixed chat | Fetch Weekly Digest |  | ## What to change<br><br>1. NPM_USERNAME in the Code nodes - set your npm username<br><br>2. Telegram chatId in the digest and milestone nodes |
| 1st of Month 6PM | Schedule Trigger | Starts monthly digest on cron |  | Fetch Monthly Digest | ## What to change<br><br>1. Cron schedules if you want different reporting times<br><br>2. NPM_USERNAME in the Code nodes - set your npm username<br><br>3. MILESTONES array to customize alert thresholds |
| Fetch Monthly Digest | Code | Builds automatic monthly digest | 1st of Month 6PM | Send Monthly Digest | ## What to change<br><br>1. Cron schedules if you want different reporting times<br><br>2. NPM_USERNAME in the Code nodes - set your npm username<br><br>3. MILESTONES array to customize alert thresholds |
| Send Monthly Digest | Telegram | Sends monthly digest to fixed chat | Fetch Monthly Digest |  | ## What to change<br><br>1. NPM_USERNAME in the Code nodes - set your npm username<br><br>2. Telegram chatId in the digest and milestone nodes |
| Milestone (daily check 9AM) | Schedule Trigger | Starts daily milestone check on cron |  | Check Milestones | ## What to change<br><br>1. Cron schedules if you want different reporting times<br><br>2. NPM_USERNAME in the Code nodes - set your npm username<br><br>3. MILESTONES array to customize alert thresholds |
| Check Milestones | Code | Detects milestone crossings based on cumulative downloads | Milestone (daily check 9AM) | Has Milestones? | ## What to change<br><br>1. Cron schedules if you want different reporting times<br><br>2. NPM_USERNAME in the Code nodes - set your npm username<br><br>3. MILESTONES array to customize alert thresholds |
| Has Milestones? | If | Sends alert only when milestones exist | Check Milestones | Send Milestone Alert | ## What to change<br><br>1. Cron schedules if you want different reporting times<br><br>2. NPM_USERNAME in the Code nodes - set your npm username<br><br>3. MILESTONES array to customize alert thresholds |
| Send Milestone Alert | Telegram | Sends milestone notification to fixed chat | Has Milestones? |  | ## What to change<br><br>1. NPM_USERNAME in the Code nodes - set your npm username<br><br>2. Telegram chatId in the digest and milestone nodes |
| Sticky Note9 | Sticky Note | Canvas documentation for workflow purpose and setup |  |  |  |
| Sticky Note10 | Sticky Note | Canvas documentation for command/report customization |  |  |  |
| Sticky Note11 | Sticky Note | Canvas documentation for schedule and milestone customization |  |  |  |
| Sticky Note | Sticky Note | Canvas example output image |  |  | ## Final Output<br>![](https://res.cloudinary.com/dewqzljou/image/upload/v1773000261/NPM_package_tracker_-_demo_a59te9.jpg) |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it something like `NPM package tracker`.

2. **Create Telegram credentials**
   - In Telegram, create a bot via BotFather.
   - Copy the bot token.
   - In n8n, create a `Telegram API` credential using that token.
   - Use the same credential for the Telegram Trigger and all Telegram send nodes.

3. **Add the interactive entry node**
   - Create a `Telegram Trigger` node.
   - Set it to listen for `message` updates.
   - Attach your Telegram credential.

4. **Add the command parser**
   - Create a `Code` node named `Parse Command`.
   - Connect `Telegram Trigger -> Parse Command`.
   - Paste JavaScript that:
     - reads `message.text`
     - reads `message.chat.id`
     - strips leading `/`
     - lowercases input
     - compares against `downloads`, `weekly`, `status`, `trending`, `help`
     - uses Levenshtein distance
     - sets:
       - `command`
       - `original`
       - `chatId`
       - `wasCorrected`
       - `correctedTo`
   - Use a threshold of `<= 4` for typo correction.

5. **Add the command router**
   - Create a `Switch` node named `Switch Command`.
   - Connect `Parse Command -> Switch Command`.
   - Configure six outputs with renamed keys:
     1. `downloads` where `{{$json.command}} == downloads`
     2. `weekly` where `{{$json.command}} == weekly`
     3. `status` where `{{$json.command}} == status`
     4. `trending` where `{{$json.command}} == trending`
     5. `help` where `{{$json.command}} == help`
     6. `unknown` where `{{$json.command}} == unknown`

6. **Add the `/downloads` report generator**
   - Create a `Code` node named `Fetch All-Time Downloads`.
   - Connect Switch output `downloads` to it.
   - In code:
     - define `NPM_USERNAME`
     - optionally include typo correction note from input
     - fetch packages from:
       `https://registry.npmjs.org/-/v1/search?text=maintainer:${NPM_USERNAME}&size=50`
     - fallback to a hardcoded package array if fetch fails
     - for each package fetch:
       `https://api.npmjs.org/downloads/point/2020-01-01:${today}/${pkg}`
     - sum totals, sort descending, format Markdown
     - return `{ message, chatId }`

7. **Add the `/downloads` Telegram sender**
   - Create a `Telegram` node named `Send Downloads`.
   - Connect `Fetch All-Time Downloads -> Send Downloads`.
   - Set:
     - Text: `{{$json.message}}`
     - Chat ID: `{{$json.chatId}}`
     - Parse Mode: `Markdown`
     - Append Attribution: off
   - Attach Telegram credential.

8. **Add the `/weekly` report generator**
   - Create a `Code` node named `Fetch Weekly Downloads`.
   - Connect Switch output `weekly`.
   - Use the same package discovery logic.
   - For each package fetch:
     `https://api.npmjs.org/downloads/point/last-week/${pkg}`
   - Build a Markdown message including a displayed 7-day range.
   - Return `{ message, chatId }`.

9. **Add the `/weekly` sender**
   - Create a `Telegram` node named `Send Weekly`.
   - Connect `Fetch Weekly Downloads -> Send Weekly`.
   - Use the same sender settings as above with dynamic `chatId`.

10. **Add the `/status` report generator**
    - Create a `Code` node named `Fetch Status`.
    - Connect Switch output `status`.
    - Use package discovery/fallback logic.
    - For each package fetch both:
      - `last-week`
      - all-time `2020-01-01:today`
    - Build a Markdown message with:
      - weekly section
      - weekly total
      - all-time section
      - grand total
    - Return `{ message, chatId }`.

11. **Add the `/status` sender**
    - Create a `Telegram` node named `Send Status`.
    - Connect `Fetch Status -> Send Status`.
    - Use dynamic `chatId`, Markdown, attribution disabled.

12. **Add the `/trending` report generator**
    - Create a `Code` node named `Fetch Trending`.
    - Connect Switch output `trending`.
    - Use package discovery/fallback logic.
    - Compute:
      - now
      - one week ago
      - two weeks ago
    - For each package fetch:
      - this week range
      - previous week range
    - Compute:
      - current week downloads
      - previous week downloads
      - delta
      - growth percentage
      - indicator emoji
    - Sort by current week downloads.
    - Return `{ message, chatId }`.

13. **Add the `/trending` sender**
    - Create a `Telegram` node named `Send Trending`.
    - Connect `Fetch Trending -> Send Trending`.
    - Use dynamic `chatId`, Markdown, attribution disabled.

14. **Add the `/help` generator**
    - Create a `Code` node named `Help Message`.
    - Connect Switch output `help`.
    - Return a static Markdown message listing supported commands.
    - Include the incoming `chatId`.

15. **Add the `/help` sender**
    - Create a `Telegram` node named `Send Help`.
    - Connect `Help Message -> Send Help`.
    - Use dynamic `chatId`, Markdown, attribution disabled.

16. **Add the unknown-command generator**
    - Create a `Code` node named `Unknown Command`.
    - Connect Switch output `unknown`.
    - Use `original` and `chatId` from input.
    - Build a Markdown message indicating the command was not recognized and list valid commands.
    - Return `{ message, chatId }`.

17. **Add the unknown-command sender**
    - Create a `Telegram` node named `Send Unknown`.
    - Connect `Unknown Command -> Send Unknown`.
    - Use dynamic `chatId`, Markdown, attribution disabled.

18. **Add the weekly scheduled trigger**
    - Create a `Schedule Trigger` node named `Every Friday 6PM`.
    - Set cron expression to `0 18 * * 5`.

19. **Add the weekly digest generator**
    - Create a `Code` node named `Fetch Weekly Digest`.
    - Connect `Every Friday 6PM -> Fetch Weekly Digest`.
    - Use package discovery/fallback logic.
    - For each package fetch:
      - current week range
      - previous week range
      - all-time range
    - Compute:
      - week total
      - previous week total
      - grand total
      - top package
      - week-over-week delta
    - Include npm profile link.
    - Return `{ message }`.

20. **Add the weekly digest sender**
    - Create a `Telegram` node named `Send Weekly Digest`.
    - Connect `Fetch Weekly Digest -> Send Weekly Digest`.
    - Set:
      - Text: `{{$json.message}}`
      - Chat ID: your fixed destination chat ID
      - Parse Mode: `Markdown`
      - Append Attribution: off

21. **Add the monthly scheduled trigger**
    - Create a `Schedule Trigger` node named `1st of Month 6PM`.
    - Set cron expression to `0 18 1 * *`.

22. **Add the monthly digest generator**
    - Create a `Code` node named `Fetch Monthly Digest`.
    - Connect `1st of Month 6PM -> Fetch Monthly Digest`.
    - Use package discovery/fallback logic.
    - Calculate:
      - last month start/end
      - previous month start/end
      - all-time range
    - For each package fetch:
      - last month downloads
      - previous month downloads
      - all-time downloads
    - Compute month totals, delta, top package, grand total.
    - Include npm profile link.
    - Return `{ message }`.

23. **Add the monthly digest sender**
    - Create a `Telegram` node named `Send Monthly Digest`.
    - Connect `Fetch Monthly Digest -> Send Monthly Digest`.
    - Set fixed Telegram destination chat ID.
    - Enable Markdown and disable attribution.

24. **Add the daily milestone trigger**
    - Create a `Schedule Trigger` node named `Milestone (daily check 9AM)`.
    - Set cron expression to `0 9 * * *`.

25. **Add the milestone checker**
    - Create a `Code` node named `Check Milestones`.
    - Connect `Milestone (daily check 9AM) -> Check Milestones`.
    - Define:
      - `NPM_USERNAME`
      - `MILESTONES = [100, 500, 1000, 5000, 10000, 50000, 100000]`
    - Use package discovery/fallback logic.
    - For each package fetch cumulative downloads up to today and yesterday.
    - If a threshold was crossed, add it to `alerts`.
    - Return:
      - `{ hasAlerts: false, message: null }` when no alerts
      - `{ hasAlerts: true, message }` when alerts exist

26. **Add the milestone filter**
    - Create an `If` node named `Has Milestones?`.
    - Connect `Check Milestones -> Has Milestones?`.
    - Condition:
      - boolean
      - `{{$json.hasAlerts}}` equals `true`

27. **Add the milestone sender**
    - Create a `Telegram` node named `Send Milestone Alert`.
    - Connect the `true` output of `Has Milestones?` to it.
    - Set:
      - Text: `{{$json.message}}`
      - Chat ID: your fixed destination chat ID
      - Parse Mode: `Markdown`
      - Append Attribution: off

28. **Update hardcoded values**
    - Replace every `NPM_USERNAME = 'monfortbrian'` with your npm maintainer username.
    - Replace every fixed `chatId = "123456789"` with the Telegram user, group, or channel ID that should receive scheduled reports.
    - If you change npm username, also update profile links such as:
      `https://www.npmjs.com/~YOUR_USERNAME`

29. **Optional: add sticky notes**
    - Add one overview note describing the workflow purpose and commands.
    - Add one note over the command/report nodes reminding users to change `NPM_USERNAME` and fixed Telegram `chatId`.
    - Add one note over the scheduled section reminding users to change cron schedules and milestone thresholds.
    - Add one note with an example output image if desired.

30. **Test the interactive branch**
    - Activate the workflow or test the Telegram Trigger.
    - Send messages such as:
      - `/downloads`
      - `weekly`
      - `/stauts` to test typo correction
      - `/help`
    - Verify responses arrive in the same chat.

31. **Test scheduled branches manually**
    - Execute `Fetch Weekly Digest`, `Fetch Monthly Digest`, and `Check Milestones` manually with test data or direct node execution.
    - Confirm Telegram send nodes work with the fixed destination chat ID.

32. **Activate the workflow**
    - Once credentials, chat IDs, username, and schedules are correct, activate the workflow.

### Credential configuration summary
- **Required credential:** Telegram API
- **Used by:**
  - Telegram Trigger
  - Send Downloads
  - Send Weekly
  - Send Status
  - Send Trending
  - Send Help
  - Send Unknown
  - Send Weekly Digest
  - Send Monthly Digest
  - Send Milestone Alert

### Input/output expectations
- **Telegram Trigger branch input:** Telegram `message` update containing `message.text` and `message.chat.id`
- **Code node outputs for on-demand commands:** must return at least `message` and `chatId`
- **Code node outputs for scheduled digests:** must return at least `message`
- **Milestone checker output:** must return `hasAlerts` and `message`

### Important implementation constraints
- Markdown formatting can break if package names or text contain special Telegram Markdown characters.
- Telegram messages have size limits; large package lists may need splitting into multiple messages.
- Package discovery is limited by `size=50`.
- All-time totals begin at `2020-01-01`, not true lifetime before that date.
- Scheduled execution time depends on the workflow/instance timezone.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Final output example image | https://res.cloudinary.com/dewqzljou/image/upload/v1773000261/NPM_package_tracker_-_demo_a59te9.jpg |
| npm maintainer profile link used in digest messages | https://www.npmjs.com/~monfortbrian |
| Main customization points: update `NPM_USERNAME` in all Code nodes and replace fixed Telegram chat IDs in scheduled send nodes | Workflow-wide |
| Cron schedules can be adjusted for weekly, monthly, and milestone checks | Workflow-wide |
| Milestone thresholds are controlled by the `MILESTONES` array in the milestone checker | `Check Milestones` node |