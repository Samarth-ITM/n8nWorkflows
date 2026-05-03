Generate UK M&A research, pitch decks and briefs from Slack using Anthropic and Google Docs/Slides

https://n8nworkflows.xyz/workflows/generate-uk-m-a-research--pitch-decks-and-briefs-from-slack-using-anthropic-and-google-docs-slides-14762


# Generate UK M&A research, pitch decks and briefs from Slack using Anthropic and Google Docs/Slides

# Workflow Analysis: Finance Research Analyst AI Employee

## 1. Workflow Overview
This workflow automates the role of a junior M&A (Mergers & Acquisitions) finance research analyst. It integrates Slack as the primary interface, using AI to classify user intents and then orchestrating a series of API calls, database operations, and document generation tasks to produce professional financial outputs.

The workflow is divided into four primary logical blocks:
- **1.1 Intake and Routing:** Captures Slack messages, filters out noise, and uses an AI classifier to determine if the user wants a "Research Pack," a "Pitch Deck," or an "Industry Briefing."
- **1.2 Research Flow:** Aggregates data from Companies House (UK), Firecrawl (Web/News), and Alpha Vantage (Market Data), then uses AI to synthesize a company profile and store it in Google Docs, PostgreSQL, and Google Sheets.
- **1.3 Pitch Flow:** Leverages previously stored research from PostgreSQL to generate a slide-ready pitch deck (via a Google Slides template) and a press release draft.
- **1.4 Briefing Flow:** Gathers macro-economic M&A data from the Office for National Statistics (ONS) and sector-specific news to create a one-page industry briefing in Google Docs.

---

## 2. Block-by-Block Analysis

### 2.1 Intake and Routing
**Overview:** Acts as the gateway, ensuring only valid user requests are processed and routing them to the correct functional pipeline.
- **Nodes Involved:** `Slack Trigger Request`, `Valid Slack Message?`, `Prepare Slack Context`, `Build Intent Request`, `Intent Classifier`, `Parse Intent`, `Intent Switch`.
- **Node Details:**
    - **Slack Trigger Request:** Webhook listener for messages in a specific channel.
    - **Valid Slack Message? (IF):** Filters out bot messages and empty text.
    - **Build Intent Request (Code):** Constructs a JSON schema for Claude-Haiku (via OpenRouter) to categorize the request into `research`, `pitch`, `brief`, or `unknown`.
    - **Intent Classifier (HTTP Request):** Calls OpenRouter API to perform the classification.
    - **Intent Switch (Switch):** Routes the workflow execution based on the `intent` variable.
- **Edge Cases:** Requests without a clear company name or sector are routed to "Unknown" or "Missing Input" responses.

### 2.2 Research Flow
**Overview:** The data-gathering engine that builds a comprehensive company dossier.
- **Nodes Involved:** `Has Research Inputs?`, `Companies House Search`, `Pick Best Company Match`, `Company Match Found?`, `Companies House Profile/Officers/Filings`, `Company News`, `Website Search`, `Alpha Vantage Overview`, `Merge Research 1-5`, `Build Research Bundle`, `Research AI`, `Parse Research Output`.
- **Node Details:**
    - **Companies House Nodes:** Uses Basic Auth to fetch official UK corporate data (profile, officers, filing history).
    - **Firecrawl Nodes:** Performs web searches for recent news and the official company website.
    - **Alpha Vantage:** Fetches stock quotes if a ticker is present.
    - **Merge Nodes:** A chain of 5 merge nodes that aggregate asynchronous API responses into a single object.
    - **Research AI (HTTP):** Uses Claude-Sonnet to transform raw JSON data into a structured research pack.
- **Failure Points:** Companies House API may not find a match for misspelled company names; Alpha Vantage fails if the company is private (handled by `No Market Data Placeholder`).

### 2.3 Research Storage & Output
**Overview:** Converts AI-generated research into permanent assets and records.
- **Nodes Involved:** `Create Research Folder`, `Create Profile Doc`, `Update Profile Doc`, `Upsert Company Record`, `Save Research Memory`, `Update Client DB Research`, `Research Response`.
- **Node Details:**
    - **Google Drive/Docs:** Creates a dedicated folder and a markdown-formatted document for the company profile.
    - **Upsert Company Record (Postgres):** Saves metadata (SIC codes, directors, market cap) into a table; updates the record if the company already exists.
    - **Save Research Memory (Postgres):** Stores the AI-generated profile text for later retrieval by the Pitch flow.
    - **Update Client DB (Sheets):** Syncs the latest research status to a master Google Sheet.

### 2.4 Pitch Flow
**Overview:** Transforms existing research into a client-facing pitch deck and press release.
- **Nodes Involved:** `Has Pitch Company?`, `Load Company Record`, `Has Prior Research?`, `Load Research Memory`, `Create Pitch Folder`, `Build Pitch Prompt`, `Pitch AI`, `Parse Pitch Output`, `Copy Slides Template`, `Create Custom Presentation`, `Create Press Release Doc`, `Update Pitch Assets`, `Update Client DB Pitch`.
- **Node Details:**
    - **Load Research Memory (Postgres):** Crucial step; retrieves the `content_text` from a previous "Research" run.
    - **Build Pitch Prompt (Code):** Instructs the AI to create *concise bullet points* suitable for slides rather than paragraphs.
    - **Copy Slides Template (Drive):** Copies a predefined Google Slides master template.
    - **Create Custom Presentation (Slides):** Uses the `replaceText` operation to swap placeholders (e.g., `{company_name}`) with AI-generated content.
- **Edge Cases:** If no prior research exists in Postgres, the workflow triggers `Pitch Missing Research Response` and stops.

### 2.5 Briefing Flow
**Overview:** Generates a macro-view of a specific industry sector.
- **Nodes Involved:** `Has Brief Sector?`, `Sector News`, `ONS M&A Source`, `Merge Brief Sources`, `Build Brief Bundle`, `Build Brief Prompt`, `Brief AI`, `Parse Brief Output`, `Create Brief Folder`, `Create Brief Doc`, `Update Brief Doc`.
- **Node Details:**
    - **Sector News (Firecrawl):** Searches for trends and M&A activity in the specified sector.
    - **ONS M&A Source (HTTP):** Scrapes the Office for National Statistics for macro UK M&A data.
    - **Brief AI:** Synthesizes sector news and ONS data into a one-page briefing.
- **Output:** A Google Doc containing the industry briefing.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Slack Trigger Request | Slack Trigger | Entry point | - | Valid Slack Message? | Intake + Routing |
| Intent Classifier | HTTP Request | AI Intent Detection | Build Intent Request | Parse Intent | Intake + Routing |
| Intent Switch | Switch | Logic Router | Parse Intent | Research/Pitch/Brief/Unknown paths | Intake + Routing |
| Companies House Search | HTTP Request | Data Acquisition | Has Research Inputs? | Pick Best Company Match | Research Flow |
| Research AI | HTTP Request | Synthesis | Build Research Prompt | Parse Research Output | Research Flow |
| Upsert Company Record | Postgres | Data Persistence | Update Profile Doc | Save Research Memory | Research Storage |
| Update Client DB Research | Google Sheets | Client Tracking | Save Research Memory | Research Response | Research Storage |
| Load Research Memory | Postgres | Context Retrieval | Has Prior Research? | Has Research Memory? | Pitch Core Flow |
| Pitch AI | HTTP Request | Material Generation | Build Pitch Prompt | Parse Pitch Output | Pitch Core Flow |
| Create Custom Presentation| Google Slides | Template Filling | Copy Slides Template | Create Press Release Doc | Pitch saving |
| Sector News | HTTP Request | Sector Analysis | Has Brief Sector? | Wrap Sector News | Briefing Flow |
| ONS M&A Source | HTTP Request | Macro Analysis | Has Brief Sector? | Wrap ONS M&A | Briefing Flow |
| Update Brief Doc | Google Docs | Document Finalization| Create Brief Doc | Brief Response | Industry briefing Store |
| Send Slack Reply | Slack | Final Notification | Response Nodes | - | Slack Replies |

---

## 4. Reproducing the Workflow from Scratch

### Step 1: Infrastructure & Credentials
Set up the following credentials in n8n:
- **Slack OAuth2:** App with `channels:history`, `chat:write`, and `groups:history` scopes.
- **OpenRouter API:** For access to Claude 3.5/4.5 models.
- **Google OAuth2:** Access to Drive, Docs, Sheets, and Slides.
- **Companies House API:** Basic Auth key.
- **Firecrawl API:** Header Auth key.
- **Alpha Vantage API:** API Key for market data.
- **PostgreSQL:** Connection to a database with tables `aiemp_research_analyst_companies` and `aiemp_research_analyst_research_memory`.

### Step 2: Intake Logic
1. Create a **Slack Trigger** node to listen for messages.
2. Add an **IF Node** to verify `bot_id` is absent and `text` is present.
3. Use a **Code Node** to clean the input and a second **Code Node** to create a JSON prompt for the **HTTP Request (OpenRouter)**.
4. Add a **Switch Node** to branch the flow based on the `intent` (research, pitch, brief, unknown).

### Step 3: Research Pipeline
1. **Search:** HTTP Request $\rightarrow$ Companies House API $\rightarrow$ Code Node (to rank and pick the best match).
2. **Data Gathering:** Create parallel HTTP requests for:
   - Companies House Profile, Officers, and Filings.
   - Firecrawl (News and Website).
   - Alpha Vantage (Price/Market Cap).
3. **Aggregation:** Chain **Merge Nodes** to combine all these outputs into one JSON object.
4. **Synthesis:** Use a **Code Node** to build a prompt $\rightarrow$ **HTTP Request (OpenRouter Claude-Sonnet)** $\rightarrow$ **Code Node** to parse JSON.

### Step 4: Storage & Document Generation
1. **Drive:** Create a folder using the Google Drive node.
2. **Docs:** Create a document $\rightarrow$ Update it with the AI's `profile_markdown`.
3. **DB:** Execute a SQL query in the Postgres node to `INSERT ... ON CONFLICT UPDATE` the company record.
4. **Sheets:** Use the Google Sheets node (`appendOrUpdate`) to sync the data.

### Step 5: Pitch & Briefing Pipelines
1. **Pitch:** 
   - Query Postgres for the `company_id` and `research_memory`.
   - Use AI to generate bullet points.
   - **Google Drive:** Copy a specific Slides template ID.
   - **Google Slides:** Use the "Replace Text" operation for `{company_name}`, `{financials}`, etc.
2. **Briefing:**
   - HTTP Request $\rightarrow$ Firecrawl (Sector News) AND HTTP Request $\rightarrow$ ONS Website.
   - Merge sources $\rightarrow$ AI Synthesis $\rightarrow$ Google Doc creation.

### Step 6: Response Loop
- For every end-branch, use a **Code Node** to format a Slack message containing the relevant Google Doc/Slide links.
- Connect all ends to a final **Slack Node** to post the reply in the original thread (`thread_ts`).

---

## 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Full Setup Video Tutorial | [YouTube Link](https://www.youtube.com/watch?v=GONk4r8lKIg) |
| Resources & Templates Folder | [Google Drive](https://drive.google.com/drive/folders/15xftVTrQLGJN0SjyEQsOM96cWVNuJgDl?usp=sharing) |
| AI Cost Warning | High-token models (Sonnet) used in Research flow; Haiku used for classification/briefs to save costs. |
| Template Dependency | The Pitch flow requires a pre-existing Google Slides template with specific placeholders. |