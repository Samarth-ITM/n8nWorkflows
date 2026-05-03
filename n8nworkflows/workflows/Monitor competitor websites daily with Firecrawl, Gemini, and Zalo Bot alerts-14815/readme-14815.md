Monitor competitor websites daily with Firecrawl, Gemini, and Zalo Bot alerts

https://n8nworkflows.xyz/workflows/monitor-competitor-websites-daily-with-firecrawl--gemini--and-zalo-bot-alerts-14815


# Monitor competitor websites daily with Firecrawl, Gemini, and Zalo Bot alerts

The user provided a workflow JSON for an n8n workflow titled "Monitor competitor websites daily with Firecrawl, Gemini, and Zalo Bot alerts". I need to produce a comprehensive, structured, self-contained reference document in English according to the specified sections: Workflow Overview, Block-by-Block Analysis, Summary Table, Reproducing the Workflow from Scratch, General Notes & Resources. I need to include sticky notes content and preserve links. I need to ensure I cover all nodes, no raw JSON (except minimal examples), professional English, no mention of IDs. Must ensure I capture the connections correctly.

First, I'll identify all nodes in the workflow:

- Sticky Note (id: sticky-main)
- Sticky Note1 (sticky-step1)
- Sticky Note2 (sticky-step2)
- Sticky Note3 (sticky-step3)
- Sticky Note4 (sticky-step4)
- Sticky Note5 (sticky-step5)
- Sticky Note6 (sticky-step6)
- Sticky Note7 (sticky-disclaimer)
- Daily 9AM Trigger (schedule trigger)
- Set Configuration (set node)
- Scrape Competitor (Firecrawl)
- AI Summarize Vietnamese (Google Gemini)
- Format Alert Message (set node)
- Send Zalo Alert (Zalo Bot)

Connections:
Daily 9AM Trigger -> Set Configuration -> Scrape Competitor -> AI Summarize Vietnamese -> Format Alert Message -> Send Zalo Alert

So a linear pipeline.

Need to describe each logical block:

Block 1: Scheduling (Daily 9AM Trigger) + config (Set Configuration). Might be considered separate blocks, but could group them as "Input Configuration & Scheduling".

Block 2: Web Scraping (Scrape Competitor node) using Firecrawl.

Block 3: AI Summarization (AI Summarize Vietnamese node) using Google Gemini.

Block 4: Message Formatting (Format Alert Message) using Set node.

Block 5: Notification (Send Zalo Alert) using Zalo Bot.

Also sticky notes: major note, per-step notes, disclaimer note.

Will need to provide detailed Node details:

- Type, technical role
- Configuration choices
- Key expressions/variables
- Input/output connections
- Edge cases or potential failure types
- Sub-workflow reference (none)

For each node, list its type.

For schedule trigger: n8n-nodes-base.scheduleTrigger, version 1.2, cron expression "0 9 * * *". Runs daily at 9 AM local time zone.

Set Configuration: n8n-nodes-base.set (type version 3.4). Assignments: competitorUrl (string), competitorName (string), zaloChatId (string). Provides config for downstream nodes.

Scrape Competitor: n8n-nodes-firecrawl-v2.firecrawl (type version 1). URL uses expression "={{ $json.competitorUrl }}". Use credentials for Firecrawl API.

AI Summarize Vietnamese: @n8n/n8n-nodes-langchain.googleGemini (type version 1). Model: gemini-2.0-flash (list). Prompt uses Vietnamese text, referencing competitorName and markdown from Firecrawl. Credentials: googlePalmApi (Gemini Pro Key). Output likely includes parts[0].text in content.

Format Alert Message: n8n-nodes-base.set (type version 3.4). Assignments: alertMessage (string). Contains template: "BAO CAO DOI THU HANG NGAY\n\nDoi thu: ...". Uses expressions for competitorName, competitorUrl, $now.format('dd/MM/yyyy HH:mm'), and AI output ($json.content.parts[0].text). Adds footer "Tu dong boi n8n + Firecrawl + Zalo Bot".

Send Zalo Alert: n8n-nodes-zalo-platform.zaloBot (type version 1). Parameters: text = $json.alertMessage, chatId = $('Set Configuration').item.json.zaloChatId. Uses Zalo Bot credentials.

Potential failure types: Firecrawl API errors, anti-bot blocks, rate limits, timeout. Gemini API errors, model output format changes, response parsing failures. Zalo Bot API errors (invalid chat ID, token). Cron schedule misconfiguration.

Will need to list sticky notes: 

- Sticky Note (main): contains description of workflow, how it works, setup steps, customization suggestions, help links. Cover nodes: all. Contains links: Firecrawl node guide (https://thenexova.com/n8n-firecrawl-node-web-scraping-crawling-and-ai-extraction-guide/), Zalo Bot node guide (https://thenexova.com/n8n-zalo-bot-node-complete-setup-and-operations-guide/).

- Sticky Note1: "Schedule daily run. The Schedule Trigger fires once per day at 9:00 AM. Adjust the cron expression to run hourly, weekly, or at a different time." Covers Daily 9AM Trigger.

- Sticky Note2: "Configure monitoring target. Holds all the variables you need to customize: competitor URL, brand name, and Zalo Chat ID. Edit this node to point at your competitor." Covers Set Configuration.

- Sticky Note3: "Scrape competitor page. Firecrawl fetches the page and returns clean markdown. Handles JavaScript-rendered SPAs, anti-bot protection, and removes navigation and ads automatically." Covers Scrape Competitor.

- Sticky Note4: "Summarize with AI. Google Gemini 2.0 Flash reads the scraped markdown and generates a concise Vietnamese summary highlighting new products, pricing changes, and latest updates." Covers AI Summarize Vietnamese.

- Sticky Note5: "Format alert message. Combines the AI summary with competitor name, URL, and timestamp into a clean notification text ready for Zalo." Covers Format Alert Message.

- Sticky Note6: "Send Zalo notification. Delivers the formatted alert to your Zalo Bot conversation. The bot token is configured in the Zalo Bot credential." Covers Send Zalo Alert.

- Sticky Note7: "Community Nodes - Self-hosted only. This template uses two community nodes (`n8n-nodes-firecrawl-v2` and `n8n-nodes-zalo-platform`) which require self-hosted n8n. They cannot be installed on n8n Cloud." Covers entire workflow, but maybe highlight that for Firecrawl and Zalo Bot nodes.

Need to map each node to sticky notes.

Now, the summary table should have columns: Node Name, Node Type, Functional Role, Input Node(s), Output Node(s), Sticky Note.

Will include all nodes (including sticky note nodes? Usually, sticky note nodes are not functional; they are annotations. But instructions say "List all workflow nodes in a single table" - we probably should include sticky note nodes as nodes, but their type is "Sticky Note". However, sticky notes have no functional connections. Provide them too.

But the sticky notes themselves are not part of the pipeline; they may have no input/output nodes. Probably put "N/A" for Input and Output.

Alternatively, include only functional nodes. The instruction: "List all workflow nodes". This could include sticky notes. We'll include them for completeness. Mark them with no input/output.

Now, produce the Block-by-Block analysis.

Define blocks:

Block 1: Scheduling and Configuration

- Overview: Initiates workflow daily at 9 AM and sets up required parameters.
- Nodes Involved: Daily 9AM Trigger, Set Configuration
- Node details for each.

Block 2: Web Scraping

- Overview: Uses Firecrawl to scrape competitor website content as markdown.
- Nodes: Scrape Competitor

Block 3: AI Summarization

- Overview: Feeds scraped markdown into Google Gemini to generate Vietnamese summary.
- Nodes: AI Summarize Vietnamese

Block 4: Alert Formatting

- Overview: Combines AI output, metadata, and timestamp into final notification message.
- Nodes: Format Alert Message

Block 5: Notification Delivery

- Overview: Sends formatted alert message to Zalo chat via Zalo Bot.
- Nodes: Send Zalo Alert

Also note the sticky notes block.

Now for reproduction steps: Provide step-by-step instruction, including installation of community nodes, setting up credentials, configuring each node.

Will need to note that the schedule trigger uses cron expression "0 9 * * *". Could adjust timezone via n8n server.

Add a note about credentials: Firecrawl API key, Google Gemini (Google AI Studio) API key, Zalo Bot token.

Potential need to create credentials in n8n: Firecrawl API credential type "firecrawlApi", Google Gemini credential "googlePalmApi", Zalo Bot credential "zaloBotApi".

Set Configuration node assignments: competitorUrl (string), competitorName (string), zaloChatId (string). Provide placeholder values.

Scrape Competitor: set URL to expression referencing competitorUrl. Use default scrapeOptions (empty object) for default behavior.

AI Summarize Vietnamese: Model selection: list, value "models/gemini-2.0-flash". Prompt: Vietnamese instruction referencing competitorName and markdown from previous node.

Format Alert Message: assign alertMessage using template. Use $now.format('dd/MM/yyyy HH:mm') for date.

Send Zalo Alert: parameters: text and chatId referencing previous node.

Add note about ensuring the chat ID is from Zalo Bot.

Also note: Use of self-hosted n8n because of community nodes.

Now produce general notes & resources: sticky note content with links, community nodes, etc. Provide table format.

Check compliance: no mention of sticky note IDs, preserve link format.

Also, need to avoid raw JSON except minimal.

Will produce structured sections.

Now, let's produce the final answer.## 1. Workflow Overview  

**Purpose** – Deliver a daily, AI‑curated Vietnamese briefing about a competitor’s website directly to a Zalo chat. The workflow automates the entire pipeline: schedule → configure → scrape → summarize → format → notify.  

**Logical blocks**  

| Block | Function | Nodes |
|-------|----------|-------|
| 1. Scheduling & Configuration | Run the workflow every day at 9 AM and provide all runtime parameters (competitor URL, brand name, Zalo chat ID). | Daily 9AM Trigger, Set Configuration |
| 2. Web Scraping | Retrieve the competitor’s page as clean markdown, handling JS‑rendered content and anti‑bot protection. | Scrape Competitor |
| 3. AI Summarization | Generate a ≤200‑word Vietnamese summary of key updates (new products, price changes, blog posts, announcements). | AI Summarize Vietnamese |
| 4. Alert Formatting | Combine the AI summary with metadata (competitor name, URL, timestamp) into a ready‑to‑send notification text. | Format Alert Message |
| 5. Notification Delivery | Push the final text to a Zalo Bot conversation. | Send Zalo Alert |

The pipeline is strictly linear: Trigger → Set → Scrape → Summarize → Format → Zalo.

---

## 2. Block‑by‑Block Analysis  

### Block 1 – Scheduling & Configuration  

**Overview**  
A schedule trigger fires once per day (9 AM). The following Set node stores all configurable values so downstream nodes can reference them without hard‑coding.

**Nodes Involved**  
- Daily 9AM Trigger  
- Set Configuration  

#### Daily 9AM Trigger  
| Property | Detail |
|----------|--------|
| **Type** | `n8n-nodes-base.scheduleTrigger` (v1.2) |
| **Technical role** | Initiates the workflow on a time‑based schedule. |
| **Configuration** | Cron expression `0 9 * * *` (runs at 09:00 server time). |
| **Input connections** | None (entry point). |
| **Output connections** | → Set Configuration (main index 0). |
| **Edge cases / failures** | – Wrong cron syntax → workflow never runs or runs at unintended times.<br>– Server timezone mismatch → alerts arrive at unexpected local time. |
| **Sticky note** | “Schedule daily run – The Schedule Trigger fires once per day at 9:00 AM. Adjust the cron expression to run hourly, weekly, or at a different time.” |

#### Set Configuration  
| Property | Detail |
|----------|--------|
| **Type** | `n8n-nodes-base.set` (v3.4) |
| **Technical role** | Provides static runtime parameters for all downstream nodes. |
| **Configuration** | Three string assignments: <br>• `competitorUrl` → default `https://thenexova.com/` <br>• `competitorName` → default `THE NEXOVA` <br>• `zaloChatId` → default `818e7bf147beaee0f7af` |
| **Expressions** | No dynamic expressions in this node; values are static placeholders meant to be edited per deployment. |
| **Input connections** | ← Daily 9AM Trigger (main index 0). |
| **Output connections** | → Scrape Competitor (main index 0). |
| **Edge cases / failures** | – Empty or malformed URL → Firecrawl fails.<br>– Invalid Zalo Chat ID → Zalo send fails.<br>– Missing assignment → downstream nodes reference undefined fields. |
| **Sticky note** | “Configure monitoring target – Holds all the variables you need to customize: competitor URL, brand name, and Zalo Chat ID. Edit this node to point at your competitor.” |

---

### Block 2 – Web Scraping  

**Overview**  
The Firecrawl community node fetches the configured competitor URL and returns the page content as markdown, automatically handling JavaScript rendering, anti‑bot protections, and noise removal (ads, navigation).

**Nodes Involved**  
- Scrape Competitor  

#### Scrape Competitor  
| Property | Detail |
|----------|--------|
| **Type** | `n8n-nodes-firecrawl-v2.firecrawl` (v1) |
| **Technical role** | Retrieve a clean markdown representation of the target web page. |
| **Configuration** | • **URL** – expression `={{ $json.competitorUrl }}` (reads from Set Configuration). <br>• **Scrape Options** – left empty (uses Firecrawl defaults). |
| **Credentials** | `Firecrawl account` (API key stored in n8n credential `firecrawlApi`). |
| **Input connections** | ← Set Configuration (main index 0). |
| **Output connections** | → AI Summarize Vietnamese (main index 0). |
| **Edge cases / failures** | – URL returns 404/5xx → node error.<br>– Anti‑bot detection overrides → Firecrawl may return empty content.<br>– Timeout on very heavy pages → increase Firecrawl timeout or reduce page size.<br>– Rate limits on Firecrawl Cloud plan → may need plan upgrade. |
| **Sticky note** | “Scrape competitor page – Firecrawl fetches the page and returns clean markdown. Handles JavaScript-rendered SPAs, anti-bot protection, and removes navigation and ads automatically.” |

---

### Block 3 – AI Summarization  

**Overview**  
Google Gemini 2.0 Flash reads the scraped markdown and produces a concise Vietnamese summary (≤200 words) focused on new products, price changes, blog posts, and important announcements.

**Nodes Involved**  
- AI Summarize Vietnamese  

#### AI Summarize Vietnamese  
| Property | Detail |
|----------|--------|
| **Type** | `@n8n/n8n-nodes-langchain.googleGemini` (v1) |
| **Technical role** | LLM‑powered summarization in Vietnamese. |
| **Configuration** | • **Model** – `models/gemini-2.0-flash` (selected via list mode). <br>• **Messages** – one system/user message containing a Vietnamese prompt: <br>`=Ban la chuyen gia phan tich doi thu canh tranh. Hay doc noi dung trang web sau tu website cua {{ $('Set Configuration').item.json.competitorName }} va tom tat ngan gon bang tieng Viet cac thong tin quan trong: san pham moi, thay doi gia, bai viet blog moi, thong bao quan trong. Toi da 200 tu. Dung bullet points.\n\nNoi dung trang web:\n{{ $json.markdown }}` |
| **Expressions** | – `$('Set Configuration').item.json.competitorName` – brand name.<br>– `$json.markdown` – scraped content from previous node. |
| **Credentials** | `Google Gemini Pro Key` (stored as `googlePalmApi`). |
| **Input connections** | ← Scrape Competitor (main index 0). |
| **Output connections** | → Format Alert Message (main index 0). |
| **Edge cases / failures** | – Gemini API quota exceeded → request fails.<br>– Response format changes (e.g., no `parts[0].text`) → downstream expression breaks.<br>– Long markdown > model context window → truncate or pre‑process content. |
| **Sticky note** | “Summarize with AI – Google Gemini 2.0 Flash reads the scraped markdown and generates a concise Vietnamese summary highlighting new products, pricing changes, and latest updates.” |

---

### Block 4 – Alert Formatting  

**Overview**  
Merges the AI‑generated summary with meta information (competitor name, URL, timestamp) into a clean notification string.

**Nodes Involved**  
- Format Alert Message  

#### Format Alert Message  
| Property | Detail |
|----------|--------|
| **Type** | `n8n-nodes-base.set` (v3.4) |
| **Technical role** | Compose the final alert text for Zalo. |
| **Configuration** | Single string assignment: <br>• **Name** – `alertMessage` <br>• **Value** – expression: <br>`=BAO CAO DOI THU HANG NGAY\n\nDoi thu: {{ $('Set Configuration').item.json.competitorName }}\nURL: {{ $('Set Configuration').item.json.competitorUrl }}\nNgay: {{ $now.format('dd/MM/yyyy HH:mm') }}\n\nTom tat cap nhat:\n{{ $json.content.parts[0].text }}\n\nTu dong boi n8n + Firecrawl + Zalo Bot` |
| **Expressions** | – Pulls `competitorName`, `competitorUrl` from Set Configuration.<br>– `$now.format('dd/MM/yyyy HH:mm')` for timestamp.<br>– `$json.content.parts[0].text` extracts the summary text from Gemini output. |
| **Input connections** | ← AI Summarize Vietnamese (main index 0). |
| **Output connections** | → Send Zalo Alert (main index 0). |
| **Edge cases / failures** | – Gemini response structure missing `content.parts[0].text` → expression error.<br>– Timezone mismatch for `$now` → adjust n8n global timezone setting. |
| **Sticky note** | “Format alert message – Combines the AI summary with competitor name, URL, and timestamp into a clean notification text ready for Zalo.” |

---

### Block 5 – Notification Delivery  

**Overview**  
Sends the assembled alert string to a Zalo Bot conversation identified by the chat ID stored earlier.

**Nodes Involved**  
- Send Zalo Alert  

#### Send Zalo Alert  
| Property | Detail |
|----------|--------|
| **Type** | `n8n-nodes-zalo-platform.zaloBot` (v1) |
| **Technical role** | Deliver the notification via Zalo’s Bot Platform. |
| **Configuration** | • **text** – expression `={{ $json.alertMessage }}` (from Format Alert Message). <br>• **chatId** – expression `={{ $('Set Configuration').item.json.zaloChatId }}` |
| **Credentials** | `Zalo Bot account` (stored as `zaloBotApi`). |
| **Input connections** | ← Format Alert Message (main index 0). |
| **Output connections** | None (terminal node). |
| **Edge cases / failures** | – Invalid or revoked chat ID → API error.<br>– Expired Zalo Bot token → re‑create or refresh credential.<br>– Zalo rate limits or message size limits → split long messages or throttle sends. |
| **Sticky note** | “Send Zalo notification – Delivers the formatted alert to your Zalo Bot conversation. The bot token is configured in the Zalo Bot credential.” |

---

### Sticky Note Overview  

| Sticky Note | Content (full) | Covered Nodes |
|-------------|----------------|---------------|
| **Sticky Note (main)** | **## Daily competitor monitoring with Firecrawl and Zalo Bot alerts**<br><br>### How it works<br>1. A Schedule Trigger runs daily at 9:00 AM (configurable).<br>2. The Set Configuration node holds the competitor URL, brand name, and Zalo Chat ID.<br>3. Firecrawl scrapes the competitor page with JavaScript rendering and anti‑bot bypass, returning clean markdown.<br>4. Google Gemini summarizes key updates in Vietnamese (max 200 words, bullet points).<br>5. The formatted alert with competitor name, URL, and timestamp is sent to your Zalo Bot conversation.<br><br>### Setup steps<br>- [ ] Self‑hosted n8n required (community nodes cannot be installed on n8n Cloud)<br>- [ ] Install community nodes `n8n-nodes-firecrawl-v2` and `n8n-nodes-zalo-platform` via **Settings > Community Nodes**<br>- [ ] Create a Firecrawl API credential (Cloud or self‑hosted)<br>- [ ] Create a Zalo Bot credential from **Zalo Bot Manager** in the Zalo app<br>- [ ] Create a Google Gemini credential (free tier from [Google AI Studio](https://aistudio.google.com))<br>- [ ] Update the **Set Configuration** node with your competitor URL, brand name, and Zalo Chat ID<br>- [ ] Activate the workflow<br><br>### Customization<br>- Add multiple competitors by looping over an array of URLs<br>- Swap Gemini for OpenAI, Claude, or any local LLM<br>- Save history to Google Sheets to track changes over time<br>- Add a diff check to only alert when content actually changes<br>- Send alerts to multiple Zalo recipients or groups<br><br>### Need help?<br>- [Firecrawl node guide](https://thenexova.com/n8n-firecrawl-node-web-scraping-crawling-and-ai-extraction-guide/)<br>- [Zalo Bot node guide](https://thenexova.com/n8n-zalo-bot-node-complete-setup-and-operations-guide/) | All workflow nodes |
| **Sticky Note1** | **## Schedule daily run**<br>The Schedule Trigger fires once per day at 9:00 AM. Adjust the cron expression to run hourly, weekly, or at a different time. | Daily 9AM Trigger |
| **Sticky Note2** | **## Configure monitoring target**<br>Holds all the variables you need to customize: competitor URL, brand name, and Zalo Chat ID. Edit this node to point at your competitor. | Set Configuration |
| **Sticky Note3** | **## Scrape competitor page**<br>Firecrawl fetches the page and returns clean markdown. Handles JavaScript‑rendered SPAs, anti‑bot protection, and removes navigation and ads automatically. | Scrape Competitor |
| **Sticky Note4** | **## Summarize with AI**<br>Google Gemini 2.0 Flash reads the scraped markdown and generates a concise Vietnamese summary highlighting new products, pricing changes, and latest updates. | AI Summarize Vietnamese |
| **Sticky Note5** | **## Format alert message**<br>Combines the AI summary with competitor name, URL, and timestamp into a clean notification text ready for Zalo. | Format Alert Message |
| **Sticky Note6** | **## Send Zalo notification**<br>Delivers the formatted alert to your Zalo Bot conversation. The bot token is configured in the Zalo Bot credential. | Send Zalo Alert |
| **Sticky Note7** | **### Community Nodes - Self‑hosted only**<br>This template uses two community nodes (`n8n-nodes-firecrawl-v2` and `n8n-nodes-zalo-platform`) which require self‑hosted n8n. They cannot be installed on n8n Cloud. | Entire workflow (especially Scrape Competitor & Send Zalo Alert) |

---

## 3. Summary Table  

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|-----------|-----------|----------------|---------------|----------------|-------------|
| Sticky Note | `n8n-nodes-base.stickyNote` | Documentation – overall workflow description, setup steps, customization ideas, help links | N/A | N/A | Main sticky note (full content above) |
| Sticky Note1 | `n8n-nodes-base.stickyNote` | Documentation – schedule explanation | N/A | N/A | “Schedule daily run – The Schedule Trigger fires once per day at 9:00 AM. Adjust the cron expression to run hourly, weekly, or at a different time.” |
| Sticky Note2 | `n8n-nodes-base.stickyNote` | Documentation – configuration explanation | N/A | N/A | “Configure monitoring target – Holds all the variables you need to customize: competitor URL, brand name, and Zalo Chat ID. Edit this node to point at your competitor.” |
| Sticky Note3 | `n8n-nodes-base.stickyNote` | Documentation – scraping explanation | N/A | N/A | “Scrape competitor page – Firecrawl fetches the page and returns clean markdown. Handles JavaScript‑rendered SPAs, anti‑bot protection, and removes navigation and ads automatically.” |
| Sticky Note4 | `n8n-nodes-base.stickyNote` | Documentation – AI summarization explanation | N/A | N/A | “Summarize with AI – Google Gemini 2.0 Flash reads the scraped markdown and generates a concise Vietnamese summary highlighting new products, pricing changes, and latest updates.” |
| Sticky Note5 | `n8n-nodes-base.stickyNote` | Documentation – alert formatting explanation | N/A | N/A | “Format alert message – Combines the AI summary with competitor name, URL, and timestamp into a clean notification text ready for Zalo.” |
| Sticky Note6 | `n8n-nodes-base.stickyNote` | Documentation – Zalo notification explanation | N/A | N/A | “Send Zalo notification – Delivers the formatted alert to your Zalo Bot conversation. The bot token is configured in the Zalo Bot credential.” |
| Sticky Note7 | `n8n-nodes-base.stickyNote` | Documentation – self‑hosted requirement disclaimer | N/A | N/A | “Community Nodes - Self‑hosted only – This template uses two community nodes (`n8n-nodes-firecrawl-v2` and `n8n-nodes-zalo-platform`) which require self‑hosted n8n. They cannot be installed on n8n Cloud.” |
| Daily 9AM Trigger | `n8n-nodes-base.scheduleTrigger` | Schedule – fire workflow daily at 09:00 | None | Set Configuration | “Schedule daily run – The Schedule Trigger fires once per day at 9:00 AM. Adjust the cron expression to run hourly, weekly, or at a different time.” |
| Set Configuration | `n8n-nodes-base.set` | Store runtime variables (URL, brand, chat ID) | Daily 9AM Trigger | Scrape Competitor | “Configure monitoring target – Holds all the variables you need to customize: competitor URL, brand name, and Zalo Chat ID. Edit this node to point at your competitor.” |
| Scrape Competitor | `n8n-nodes-firecrawl-v2.firecrawl` | Web scraping – fetch page as markdown | Set Configuration | AI Summarize Vietnamese | “Scrape competitor page – Firecrawl fetches the page and returns clean markdown. Handles JavaScript‑rendered SPAs, anti‑bot protection, and removes navigation and ads automatically.” |
| AI Summarize Vietnamese | `@n8n/n8n-nodes-langchain.googleGemini` | LLM summarization in Vietnamese | Scrape Competitor | Format Alert Message | “Summarize with AI – Google Gemini 2.0 Flash reads the scraped markdown and generates a concise Vietnamese summary highlighting new products, pricing changes, and latest updates.” |
| Format Alert Message | `n8n-nodes-base.set` | Assemble final notification text | AI Summarize Vietnamese | Send Zalo Alert | “Format alert message – Combines the AI summary with competitor name, URL, and timestamp into a clean notification text ready for Zalo.” |
| Send Zalo Alert | `n8n-nodes-zalo-platform.zaloBot` | Deliver alert via Zalo Bot | Format Alert Message | None | “Send Zalo notification – Delivers the formatted alert to your Zalo Bot conversation. The bot token is configured in the Zalo Bot credential.” |

---

## 4. Reproducing the Workflow from Scratch  

Below is a step‑by‑step guide to recreate the workflow on a **self‑hosted n8n** instance. The instructions assume you have admin access to install community nodes and create credentials.

1. **Install Required Community Nodes**  
   - Navigate to **Settings → Community Nodes**.  
   - Install `n8n-nodes-firecrawl-v2`.  
   - Install `n8n-nodes-zalo-platform`.  
   - Restart n8n if prompted.

2. **Create API Credentials**  
   - **Firecrawl** – In **Credentials**, add a **Firecrawl API** credential. Enter the API key from your Firecloud Cloud account or self‑hosted instance.  
   - **Google Gemini** – In **Credentials**, add a **Google Gemini (Google PaLM API)** credential. Use an API key obtained from [Google AI Studio](https://aistudio.google.com).  
   - **Zalo Bot** – In **Credentials**, add a **Zalo Bot API** credential. Provide the Bot Token generated in the Zalo Bot Manager app.

3. **Create a New Workflow**  
   - Click **Add Workflow** → name it `Daily competitor monitoring with Firecrawl and Zalo Bot alerts`.

4. **Add and Configure the Schedule Trigger**  
   - Add a **Schedule Trigger** node.  
   - Set **Rule** → **Cron Expression** to `0 9 * * *`.  
   - Adjust the expression if you want a different schedule.  
   - (Optional) Add a sticky note: “Schedule daily run – The Schedule Trigger fires once per day at 9:00 AM. Adjust the cron expression to run hourly, weekly, or at a different time.”

5. **Add the Set Configuration Node**  
   - Add a **Set** node (type `n8n-nodes-base.set`, version 3.4).  
   - In **Assignments**, add three string fields:  
     - `competitorUrl` – default `https://thenexova.com/` (replace with your target).  
     - `competitorName` – default `THE NEXOVA` (replace with brand name).  
     - `zaloChatId` – default `818e7bf147beaee0f7af` (replace with your Zalo chat ID).  
   - Connect **Schedule Trigger → Set Configuration**.  
   - (Optional) Add sticky note: “Configure monitoring target – Holds all the variables you need to customize: competitor URL, brand name, and Zalo Chat ID. Edit this node to point at your competitor.”

6. **Add the Firecrawl Scrape Node**  
   - Add a **Firecrawl** node (type `n8n-nodes-firecrawl-v2.firecrawl`, version 1).  
   - Set **URL** to the expression `={{ $json.competitorUrl }}`.  
   - Leave **Scrape Options** empty (use defaults).  
   - Select the **Firecrawl API** credential created earlier.  
   - Connect **Set Configuration → Scrape Competitor**.  
   - (Optional) Add sticky note: “Scrape competitor page – Firecrawl fetches the page and returns clean markdown. Handles JavaScript‑rendered SPAs, anti‑bot protection, and removes navigation and ads automatically.”

7. **Add the AI Summarization Node**  
   - Add a **Google Gemini** node (type `@n8n/n8n-nodes-langchain.googleGemini`, version 1).  
   - **Model** – select **List** → choose `models/gemini-2.0-flash`.  
   - **Messages** – add one message with **Content** set to the following expression (Vietnamese prompt):  

     ```
     =Ban la chuyen gia phan tich doi thu canh tranh. Hay doc noi dung trang web sau tu website cua {{ $('Set Configuration').item.json.competitorName }} va tom tat ngan gon bang tieng Viet cac thong tin quan trong: san pham moi, thay doi gia, bai viet blog moi, thong bao quan trong. Toi da 200 tu. Dung bullet points.

     Noi dung trang web:
     {{ $json.markdown }}
     ```  

   - Select the **Google Gemini Pro Key** credential.  
   - Connect **Scrape Competitor → AI Summarize Vietnamese**.  
   - (Optional) Add sticky note: “Summarize with AI – Google Gemini 2.0 Flash reads the scraped markdown and generates a concise Vietnamese summary highlighting new products, pricing changes, and latest updates.”

8. **Add the Format Alert Message Node**  
   - Add another **Set** node (same type as step 5).  
   - **Assignments** – single string field named `alertMessage` with value:  

     ```
     =BAO CAO DOI THU HANG NGAY

     Doi thu: {{ $('Set Configuration').item.json.competitorName }}
     URL: {{ $('Set Configuration').item.json.competitorUrl }}
     Ngay: {{ $now.format('dd/MM/yyyy HH:mm') }}

     Tom tat cap nhat:
     {{ $json.content.parts[0].text }}

     Tu dong boi n8n + Firecrawl + Zalo Bot
     ```  

   - Connect **AI Summarize Vietnamese → Format Alert Message**.  
   - (Optional) Add sticky note: “Format alert message – Combines the AI summary with competitor name, URL, and timestamp into a clean notification text ready for Zalo.”

9. **Add the Zalo Bot Send Node**  
   - Add a **Zalo Bot** node (type `n8n-nodes-zalo-platform.zaloBot`, version 1).  
   - Set **text** to `={{ $json.alertMessage }}`.  
   - Set **chatId** to `={{ $('Set Configuration').item.json.zaloChatId }}`.  
   - Select the **Zalo Bot account** credential created earlier.  
   - Connect **Format Alert Message → Send Zalo Alert**.  
   - (Optional) Add sticky note: “Send Zalo notification – Delivers the formatted alert to your Zalo Bot conversation. The bot token is configured in the Zalo Bot credential.”

10. **Optional – Add Documentation Sticky Notes**  
    - Add a main sticky note summarizing the whole workflow (copy the content from the original).  
    - Add a disclaimer sticky note: “Community Nodes - Self‑hosted only – This template uses two community nodes (`n8n-nodes-firecrawl-v2` and `n8n-nodes-zalo-platform`) which require self‑hosted n8n. They cannot be installed on n8n Cloud.”

11. **Test the Workflow**  
    - Click **Execute Workflow** to verify each step.  
    - Check that Firecrawl returns markdown, Gemini produces a Vietnamese summary, the alert text is correctly formatted, and Zalo receives the message.  
    - Adjust the `competitorUrl`, `competitorName`, and `zaloChatId` values as needed.

12. **Activate the Workflow**  
    - Toggle **Active** to enable the daily schedule.  
    - Confirm the n8n server timezone aligns with the desired 9 AM local time.

---

## 5. General Notes & Resources  

| Note Content | Context or Link |
|--------------|-----------------|
| Self‑hosted n8n is required because the two community nodes cannot be installed on n8n Cloud. | Template disclaimer |
| Firecrawl node guide – detailed configuration, advanced options, and troubleshooting tips. | https://thenexova.com/n8n-firecrawl-node-web-scraping-crawling-and-ai-extraction-guide/ |
| Zalo Bot node guide – bot creation, token setup, message sending capabilities. | https://thenexova.com/n8n-zalo-bot-node-complete-setup-and-operations-guide/ |
| Free Google Gemini API key can be obtained from Google AI Studio (limited free tier). | https://aistudio.google.com |
| To monitor multiple competitors, replace the static Set Configuration with a loop (e.g., Split In Batches) over an array of URLs. | Customization suggestion |
| Change LLM provider by swapping the Google Gemini node for an OpenAI or Anthropic Claude node, adjusting the prompt and output mapping accordingly. | Customization suggestion |
| Persist historical alerts by adding a Google Sheets or Airtable node after Format Alert Message. | Customization suggestion |
| Add a diff‑check node (e.g., compare current markdown with previous version) to trigger alerts only on actual changes. | Customization suggestion |
| Zalo message size limit is roughly 2 000 characters; if alerts exceed this, split them across multiple messages. | Zalo Bot platform limitation |
| The `$now.format('dd/MM/yyyy HH:mm')` expression uses the n8n server timezone. Set `GENERIC_TIMEZONE` env var to align with your locale. | n8n configuration note |
| Firecrawl Cloud plan free tier offers limited concurrent requests; consider self‑hosting for high‑frequency scraping. | Firecrawl usage note |
| Ensure the Zalo Bot is verified and the recipient has previously interacted with the bot (required for proactive messages). | Zalo Bot messaging rule |