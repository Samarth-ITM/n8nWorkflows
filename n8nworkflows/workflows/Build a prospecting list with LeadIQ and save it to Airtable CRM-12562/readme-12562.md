Build a prospecting list with LeadIQ and save it to Airtable CRM

https://n8nworkflows.xyz/workflows/build-a-prospecting-list-with-leadiq-and-save-it-to-airtable-crm-12562


# Build a prospecting list with LeadIQ and save it to Airtable CRM

## 1. Workflow Overview

**Purpose:** This workflow converts a free-text lead search request (via n8n Chat) into LeadIQ GraphQL searches to (1) find matching people, (2) enrich their company with web data, and (3) upsert the company (Account) and person (Contact) into an Airtable “Sales CRM” style base. A second, separate block can enrich existing Airtable contacts by searching LeadIQ for email/phone data.

**Target use cases**
- Quickly building a prospecting list from a natural-language prompt (e.g., “CMOs at IT companies in New York, 11–50 employees, using HubSpot”).
- Saving results into an Airtable CRM with proper Account ↔ Contact linking.
- Optional enrichment pass to fetch emails/phones for Airtable contacts in a specific campaign.

### 1.1 Entry & User Intent Capture (Chat)
Receives the user prompt from an n8n chat interface.

### 1.2 AI Filter Extraction (Contact + Company)
Two LLM chains extract structured filters from the prompt (contact-level and company-level), then a “summary agent” merges them into a single structured object.

### 1.3 LeadIQ Search (Find People)
Builds a LeadIQ GraphQL query (`flatAdvancedSearch`) from the AI filters and calls LeadIQ.

### 1.4 Web Enrichment (Company LinkedIn/Description/HQ Address)
An agent uses Tavily (web search) to fill missing company details and outputs strict JSON.

### 1.5 Airtable Upserts (Account + Contact linking)
Upserts the Account in Airtable, retrieves the created/updated record, formats the Account link as an array, then upserts a Contact linked to that Account.

### 1.6 Optional Step #2: Enrich Existing Airtable Contacts with LeadIQ (Email/Phone)
Independently searches Airtable Contacts by Campaign, builds a LeadIQ `searchPeople` query (via LinkedIn URL), calls LeadIQ, then updates the Airtable Contact.

---

## 2. Block-by-Block Analysis

### Block 1 — Entry & Prompt Reception
**Overview:** Starts the workflow when a chat message is received and passes the prompt into the AI extraction pipeline.  
**Nodes involved:**  
- When chat message received

#### Node: When chat message received
- **Type / role:** `@n8n/n8n-nodes-langchain.chatTrigger` — public chat entrypoint (webhook-based).
- **Configuration choices:**
  - Public chat enabled.
  - Chat UI title: “LeadIQ Lead Search Agent”.
  - Placeholder example prompt.
  - Initial message greeting.
- **Key variables/expressions:** Outputs `chatInput` used downstream.
- **Connections:**
  - **Out:** to **Contact-Level Search Criteria** (main).
- **Potential failures / edge cases:**
  - If run outside chat context, `chatInput` may be missing.
  - Public endpoint can be abused; consider auth or rate limiting if exposed.

---

### Block 2 — AI Filter Extraction (Contact + Company + Summary)
**Overview:** Uses Mistral models to turn the user’s natural language prompt into structured filters compatible with LeadIQ search inputs.  
**Nodes involved:**  
- Mistral Small Latest #1
- Contact-Level Search Criteria
- Mistral Small Latest #2
- Company-level data preparation
- Mistral Small Latest #3
- Open Mistral 7B
- Structured Output Parser
- Filters Summary Agent
- Sticky Note3 (commentary)

#### Node: Mistral Small Latest #1
- **Type / role:** `lmChatMistralCloud` — LLM for contact filter extraction chain.
- **Configuration choices:** Model `mistral-small-latest`.
- **Connections:**
  - **Out (ai_languageModel):** to **Contact-Level Search Criteria**.
- **Failures / edge cases:** Invalid/expired Mistral credentials; model availability.

#### Node: Contact-Level Search Criteria
- **Type / role:** `chainLlm` — extracts contact filters (job titles, role, seniority, locations) from `chatInput`.
- **Configuration choices:**
  - Strict JSON-only output; allowed enumerations for `role` and `seniority`.
  - Omits fields if no values found (per prompt).
- **Key expressions:**
  - Uses `{{ $json.chatInput }}` (coming from Chat Trigger item).
- **Connections:**
  - **In:** from **When chat message received**
  - **Out:** to **Company-level data preparation**
- **Failures / edge cases:**
  - Model may output non-JSON or include extra text; downstream blocks assume valid JSON text.
  - Location rule says ignore “US/Europe”; user input like “US” will yield no location.

#### Node: Mistral Small Latest #2
- **Type / role:** `lmChatMistralCloud` — LLM for company filter extraction chain.
- **Configuration choices:** Model `mistral-small-latest`.
- **Connections:**  
  - **Out (ai_languageModel):** to **Company-level data preparation**.
- **Failures / edge cases:** Same as above.

#### Node: Company-level data preparation
- **Type / role:** `chainLlm` — extracts LeadIQ-compatible `companyFilter` fields (employee ranges, industries, revenue, funding, locations, keywords, technologies/categories).
- **Configuration choices:**
  - Very strict enumerations (industry list, revenue buckets, employee size buckets).
  - Funding date ranges require timestamps; “current timestamp” instruction is given to the model.
- **Key expressions:**
  - `{{ $('When chat message received').item.json.chatInput }}`
- **Connections:**
  - **In:** from **Contact-Level Search Criteria**
  - **Out:** to **Filters Summary Agent**
- **Failures / edge cases:**
  - The industry list contains typos (e.g., “IT Servies…”, “Sospitals…”). Because the prompt says “use only values from the predefined list”, outputs may include those exact typos; that may or may not match LeadIQ’s real accepted values.
  - Timestamps: the model may not reliably output correct epoch milliseconds without tools.

#### Node: Mistral Small Latest #3
- **Type / role:** `lmChatMistralCloud` — LLM powering the Filters Summary Agent.
- **Configuration choices:** Model `mistral-small-latest`.
- **Connections:**  
  - **Out (ai_languageModel):** to **Filters Summary Agent**.

#### Node: Open Mistral 7B
- **Type / role:** `lmChatMistralCloud` — LLM connected to the structured output parser.
- **Configuration choices:** Model `open-mistral-7b`.
- **Connections:**
  - **Out (ai_languageModel):** to **Structured Output Parser**.
- **Important:** This connection is unusual: the Structured Output Parser is attached to the Filters Summary Agent as an output parser, but the LLM feeding the parser is “Open Mistral 7B”. In practice, ensure the Filters Summary Agent is actually using the intended model; otherwise parsing may not match.

#### Node: Structured Output Parser
- **Type / role:** `outputParserStructured` — enforces a JSON schema for agent output.
- **Configuration choices:**
  - `autoFix: true` (attempts to fix minor JSON issues).
  - Schema example includes `contactFilter` + `companyFilter` (note singular names).
- **Connections:**
  - **Out (ai_outputParser):** to **Filters Summary Agent**.
- **Failures / edge cases:**
  - Sticky note indicates there may be a parser “error” due to GraphQL schema mismatch but it “still works”.
  - If agent output keys don’t match expected schema, downstream code may break.

#### Node: Filters Summary Agent
- **Type / role:** `agent` — merges contact-level and company-level extracted filters into one object and (intended) conforms to the Structured Output Parser.
- **Configuration choices:**
  - System message: “You are a LeadIQ search filter builder.”
  - Takes:
    - Contact-level input from `{{ $('Contact-Level Search Criteria').item.json.text }}`
    - Company-level input from `{{ $json.text }}` (company chain output)
  - Output format in prompt says:
    ```json
    { "contactFilters": {}, "companyFilters": {} }
    ```
    (plural keys)
- **Connections:**
  - **In:** from **Company-level data preparation** (main), plus LLM and output parser connections.
  - **Out:** to **GraphQL Query #1**
- **Critical edge case (likely bug):**
  - **GraphQL Query #1 expects** `agentOutput = $('Filters Summary Agent').first().json.output; const { contactFilter, companyFilter } = agentOutput;`
  - But this agent prompt suggests output keys `contactFilters` and `companyFilters` (plural), and the parser example uses `contactFilter`/`companyFilter` (singular).
  - If the agent outputs plural keys, `contactFilter`/`companyFilter` will be `undefined` and LeadIQ query will be missing filters.
  - Fix by aligning names in all three places: agent prompt, output parser schema, and GraphQL Query #1 code.

---

### Block 3 — LeadIQ “Find People” (GraphQL)
**Overview:** Converts merged filters into a LeadIQ `flatAdvancedSearch` GraphQL query and executes it.  
**Nodes involved:**  
- Sticky Note (Manage Number of Leads)
- GraphQL Query #1
- LeadIQ Database (Find People)

#### Node: GraphQL Query #1
- **Type / role:** `code` — builds the GraphQL query payload for LeadIQ `flatAdvancedSearch`.
- **Configuration choices / logic:**
  - Reads output from **Filters Summary Agent** and builds `input` with:
    - `contactFilter.titles`, `roles`, `seniorities`
    - `companyFilter.industries`
    - `companyFilter.locations` transformed into objects `{city, areaLevel1, country}` via `parseLocation()`
    - `companyFilter.employeeCountRange` mapped into `companyFilter.sizes: [{min,max}]`
    - `descriptionKeywords`, `revenueRanges`, `fundingInfoFilters`
  - Explicitly **skips technology filters** (comment says they don’t work correctly in LeadIQ).
  - Hard-coded pagination:
    - `input.limit = 1`
    - `input.skip = 0`
- **Key expressions/variables:**
  - `$('Filters Summary Agent').first().json.output`
  - Country mapping normalizes “US/USA/UK” etc.
- **Connections:**
  - **Out:** to **LeadIQ Database (Find People)**
- **Failures / edge cases:**
  - If agent output is not present / wrong key names, filters become empty.
  - `parseLocation()` expects “City, Country” (or “City, State, Country”); other formats lead to incomplete location objects.
  - **Sticky note instruction:** change `input.limit = 1` to spend more LeadIQ credits per run.

#### Node: LeadIQ Database (Find People)
- **Type / role:** `httpRequest` — POST to LeadIQ GraphQL endpoint.
- **Configuration choices:**
  - URL `https://api.leadiq.com/graphql`
  - Sends JSON body from incoming item: `jsonBody: {{$json}}`
  - Headers:
    - `Authorization: Basic <your API here>`
    - `Content-Type: application/json`
- **Connections:**
  - **In:** from **GraphQL Query #1**
  - **Out:** to **Web Enrichment Agent**
- **Failures / edge cases:**
  - Auth errors (invalid “Secret Base64 API key”).
  - LeadIQ schema changes can break the query.
  - If `people[]` is empty, downstream nodes reference `[0]` and will fail (or yield `undefined`).

---

### Block 4 — Web Enrichment (Tavily + Agent) and JSON Extraction
**Overview:** Enriches the found company with LinkedIn URL, description, and HQ address via web search, enforcing a strict JSON output, then parses it to clean JSON.  
**Nodes involved:**  
- Sticky Note4 (Web Search)
- Mistral Medium Latest
- Tavily Tool
- Web Enrichment Agent
- Output Divider (Three Parts)

#### Node: Mistral Medium Latest
- **Type / role:** `lmChatMistralCloud` — LLM for web enrichment agent.
- **Configuration choices:** Model `mistral-medium-latest`.
- **Connections:**
  - **Out (ai_languageModel):** to **Web Enrichment Agent**

#### Node: Tavily Tool
- **Type / role:** `tavilyTool` — web search tool available to the agent.
- **Configuration choices:**
  - Query uses `$fromAI('Query', ...)` (agent-provided tool input).
- **Connections:**
  - **Out (ai_tool):** to **Web Enrichment Agent**
- **Failures / edge cases:**
  - Tavily API key missing/invalid.
  - Search results may be sparse; agent instructed to return `null` for unknowns.

#### Node: Web Enrichment Agent
- **Type / role:** `agent` — uses Tavily to find company LinkedIn, description, HQ address.
- **Configuration choices:**
  - `onError: continueErrorOutput` (workflow continues even on errors).
  - Strict JSON-only output with keys:
    - `company_linkedin_url`
    - `company_description`
    - `company_hq_address`
  - Uses LeadIQ people[0].company.* fields via expressions.
- **Connections:**
  - **In:** from **LeadIQ Database (Find People)** (main), plus model/tool connections.
  - **Out:** to **Output Divider (Three Parts)**
- **Failures / edge cases:**
  - If LeadIQ response has no `people[0]`, all expressions resolve to `undefined` and prompt quality degrades.
  - Agent may still output invalid JSON; next node does `JSON.parse()` and can hard-fail.

#### Node: Output Divider (Three Parts)
- **Type / role:** `code` — normalizes the agent output into a JSON object.
- **Configuration choices / logic:**
  - Reads `text` or `output` or raw JSON; strips ```json fences; parses JSON.
  - Outputs a single item with `json: parsed`.
- **Connections:**
  - **Out:** to **Airtable: Create Account**
- **Failures / edge cases:**
  - Any non-JSON output causes `JSON.parse()` to throw and stop this branch.

---

### Block 5 — Airtable Upserts (Account then Contact)
**Overview:** Upserts the Account using LeadIQ + enrichment fields, fetches the upserted record, formats account link as an array, and upserts the Contact linked to that Account.  
**Nodes involved:**  
- Airtable: Create Account
- Get Record
- Return an Array for Account
- Add Contact
- Sticky Note7 (Airtable Note)

#### Node: Airtable: Create Account
- **Type / role:** `airtable` — upsert into Accounts table.
- **Configuration choices:**
  - Operation: **Upsert**
  - Matching column: `id`
  - Maps fields from:
    - LeadIQ: company name, employeeCount, industry, domain
    - Web enrichment output: `company_hq_address`, `company_description`, `company_linkedin_url`
  - `id` is set to LeadIQ person’s company/person id:  
    `{{ $('LeadIQ Database (Find People)').item.json.data.flatAdvancedSearch.people[0].id }}`
- **Connections:**
  - **In:** from **Output Divider (Three Parts)**
  - **Out:** to **Get Record**
- **Failures / edge cases:**
  - Airtable base/table names are placeholders (`<Name of your Base>`) and must be set.
  - If LeadIQ response is empty, expressions referencing `[0]` break.
  - Upsert key `id` must exist in Airtable schema and be consistent (string).

#### Node: Get Record
- **Type / role:** `airtable` — retrieves the Account record by Airtable record ID.
- **Configuration choices:**
  - Operation: Get
  - Record ID: `{{ $json.id }}` (from previous Airtable node output, i.e., Airtable record id)
- **Connections:**
  - **In:** from **Airtable: Create Account**
  - **Out:** to **Return an Array for Account**
- **Failures / edge cases:** Invalid record id, Airtable permission issues.

#### Node: Return an Array for Account
- **Type / role:** `code` — formats the Account link field as an array (Airtable linked record requirement).
- **Configuration choices / logic:**
  - Produces:
    ```js
    { account_link: [ $input.first().json.id ] }
    ```
- **Connections:**
  - **In:** from **Get Record**
  - **Out:** to **Add Contact**
- **Edge cases:** If account record ID missing, contact linking will fail.

#### Node: Add Contact
- **Type / role:** `airtable` — upsert into Contacts table.
- **Configuration choices:**
  - Operation: **Upsert**
  - Matching column: `id`
  - Sets:
    - `id` = Airtable Account record id (note: this is unusual; typically contact should have its own id)
    - `Name` from LeadIQ person first/last
    - `Title`, `LinkedIn` from LeadIQ
    - `Account` linked field uses `{{ $json.account_link }}`
- **Connections:**
  - **In:** from **Return an Array for Account**
- **Failures / edge cases (important):**
  - **Potential data model bug:** Contact `id` is set to `{{ $('Airtable: Create Account').item.json.id }}` (an Airtable Account record id). This makes contact IDs collide per account and prevents multiple contacts per account. Usually you’d use LeadIQ person id or LinkedIn URL as contact id.
  - If LeadIQ person array empty, name/title/linkedin are missing.

---

### Block 6 — Step #2 (Optional): Enrich Existing Airtable Contacts with LeadIQ Email/Phone
**Overview:** Finds Airtable contacts by Campaign, queries LeadIQ by LinkedIn URL via `searchPeople`, and updates Airtable contact with phone/title (and placeholder email mapping). This block is **not connected** to the chat trigger; it runs when executed starting at the Airtable search node or via another trigger you add.  
**Nodes involved:**  
- Sticky Note2 (Step #2 description)
- Search and Filter Records by Campaign
- GraphQL Query #2
- LeadIQ Database (Find Email)
- Update Record

#### Node: Search and Filter Records by Campaign 
- **Type / role:** `airtable` — searches Contacts by formula.
- **Configuration choices:**
  - Operation: Search
  - `filterByFormula`: `{Campaign} = "your campaign column data"`
  - Sort configured (by “Opportunities” field).
- **Connections:**
  - **Out:** to **GraphQL Query #2**
- **Failures / edge cases:**
  - Requires a `Campaign` field in Airtable Contacts.
  - Formula must match Airtable syntax and exact campaign value.

#### Node: GraphQL Query #2
- **Type / role:** `code` — builds LeadIQ `searchPeople` query per Airtable contact.
- **Configuration choices / logic:**
  - Encodes LinkedIn URL: `encodeURI(rawUrl)`
  - Uses `limit = 1`
  - GraphQL operation: `SearchPeople($input: SearchPeopleInput!)` with filter `linkedinUrl`.
- **Key expressions/variables:** reads from Airtable search results: `item.json.LinkedIn`
- **Connections:**
  - **Out:** to **LeadIQ Database (Find Email)**
- **Failures / edge cases:**
  - If Airtable LinkedIn field is empty, query uses empty string and results are unreliable.
  - `encodeURI()` won’t fix malformed LinkedIn URLs.

#### Node: LeadIQ Database (Find Email)
- **Type / role:** `httpRequest` — POST to LeadIQ GraphQL endpoint for email/phone enrichment.
- **Configuration choices:**
  - Same endpoint/headers as Find People.
  - `retryOnFail: true`
- **Connections:**
  - **Out:** to **Update Record**
- **Failures / edge cases:** credit usage, rate limits, auth issues, schema changes.

#### Node: Update Record
- **Type / role:** `airtable` — updates the Airtable Contact with enrichment results.
- **Configuration choices:**
  - Operation: Update
  - Record ID: `{{ $('Search and Filter Records by Campaign ').item.json.id }}`
  - Maps:
    - `Phone`: `{{ $json.data.searchPeople.results[0].personalPhones[0].value }}`
    - `Title`: `{{ $json.data.searchPeople.results[0].currentPositions[4].title }}`
    - `Email`: set to `"="` (looks like a placeholder; not actually using `personalEmails`)
- **Failures / edge cases (important):**
  - Indexing issues:
    - `results[0]` may not exist.
    - `personalPhones[0]` may not exist.
    - `currentPositions[4]` is very likely out of range; should usually be `[0]`.
  - Email is not populated from `personalEmails`; likely incomplete implementation.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| When chat message received | LangChain Chat Trigger | Public chat entrypoint | — | Contact-Level Search Criteria | ## Step #1: Lead Research & List Building<br>**Use this workflow** to build a list of leads with **LeadIQ** and save them to **Airtable**. Brown notes explain specific nodes and nuances.<br>## Start Here<br>Start the workflow by entering a prompt into the chat - describe which leads you want to find. |
| Mistral Small Latest #1 | Mistral Cloud Chat Model | LLM for contact filter extraction | — | Contact-Level Search Criteria | ## Step #1: Lead Research & List Building<br>**Use this workflow** to build a list of leads with **LeadIQ** and save them to **Airtable**. Brown notes explain specific nodes and nuances. |
| Contact-Level Search Criteria | LangChain Chain LLM | Extract contact-level filters (titles/role/seniority/location) | When chat message received | Company-level data preparation | ## Step #1: Lead Research & List Building<br>**Use this workflow** to build a list of leads with **LeadIQ** and save them to **Airtable**. Brown notes explain specific nodes and nuances. |
| Mistral Small Latest #2 | Mistral Cloud Chat Model | LLM for company filter extraction | — | Company-level data preparation | ## Step #1: Lead Research & List Building<br>**Use this workflow** to build a list of leads with **LeadIQ** and save them to **Airtable**. Brown notes explain specific nodes and nuances. |
| Company-level data preparation | LangChain Chain LLM | Extract company-level filters (industry/size/revenue/etc.) | Contact-Level Search Criteria | Filters Summary Agent | ## Step #1: Lead Research & List Building<br>**Use this workflow** to build a list of leads with **LeadIQ** and save them to **Airtable**. Brown notes explain specific nodes and nuances. |
| Mistral Small Latest #3 | Mistral Cloud Chat Model | LLM for filters summary agent | — | Filters Summary Agent | ## Step #1: Lead Research & List Building<br>**Use this workflow** to build a list of leads with **LeadIQ** and save them to **Airtable**. Brown notes explain specific nodes and nuances.<br>## Search Filters Summary<br>This agent combines filters from the contact and company data agents to build a GraphQL query. |
| Open Mistral 7B | Mistral Cloud Chat Model | LLM attached to structured output parser | — | Structured Output Parser |  |
| Structured Output Parser | LangChain Structured Output Parser | Enforce schema / auto-fix JSON | Open Mistral 7B | Filters Summary Agent |  |
| Filters Summary Agent | LangChain Agent | Merge contact + company filters | Company-level data preparation | GraphQL Query #1 | ## Step #1: Lead Research & List Building<br>**Use this workflow** to build a list of leads with **LeadIQ** and save them to **Airtable**. Brown notes explain specific nodes and nuances.<br>## Search Filters Summary<br>This agent combines filters from the contact and company data agents to build a GraphQL query. |
| GraphQL Query #1 | Code | Build LeadIQ `flatAdvancedSearch` query payload | Filters Summary Agent | LeadIQ Database (Find People) | ## Step #1: Lead Research & List Building<br>**Use this workflow** to build a list of leads with **LeadIQ** and save them to **Airtable**. Brown notes explain specific nodes and nuances.<br>## Manage Number of Leads<br>Set the number of LeadIQ credits you want to spend per single run:<br>**Change the code line**:  <input.limit = 1><br>(enter your number instead) |
| LeadIQ Database (Find People) | HTTP Request | Call LeadIQ GraphQL to find people | GraphQL Query #1 | Web Enrichment Agent | ## Step #1: Lead Research & List Building<br>**Use this workflow** to build a list of leads with **LeadIQ** and save them to **Airtable**. Brown notes explain specific nodes and nuances. |
| Mistral Medium Latest | Mistral Cloud Chat Model | LLM for enrichment agent | — | Web Enrichment Agent | ## Web Search<br>This agent finds missing company data on the internet. |
| Tavily Tool | Tavily Tool | Web search tool for enrichment | — | Web Enrichment Agent | ## Web Search<br>This agent finds missing company data on the internet. |
| Web Enrichment Agent | LangChain Agent | Find company LinkedIn/description/HQ via web | LeadIQ Database (Find People) | Output Divider (Three Parts) | ## Web Search<br>This agent finds missing company data on the internet. |
| Output Divider (Three Parts) | Code | Strip fences + JSON.parse enrichment output | Web Enrichment Agent | Airtable: Create Account |  |
| Airtable: Create Account | Airtable | Upsert Account in Airtable | Output Divider (Three Parts) | Get Record |  |
| Get Record | Airtable | Fetch Airtable Account record | Airtable: Create Account | Return an Array for Account |  |
| Return an Array for Account | Code | Format linked Account field as array | Get Record | Add Contact | ## Airtable Note<br>Account should be an array, even if it's a single record. To add a contact you have to obtaine ID of it's account in array format |
| Add Contact | Airtable | Upsert Contact linked to Account | Return an Array for Account | — | ## Airtable Note<br>Account should be an array, even if it's a single record. To add a contact you have to obtaine ID of it's account in array format |
| Search and Filter Records by Campaign  | Airtable | Find contacts to enrich (Step #2) | — | GraphQL Query #2 | ## Step #2: Enrichment: Additionally Search for Email<br>**Use this workflow** to enrich leads with web search and **LeadIQ email** (if available). |
| GraphQL Query #2 | Code | Build LeadIQ `searchPeople` query by LinkedIn URL | Search and Filter Records by Campaign  | LeadIQ Database (Find Email) | ## Step #2: Enrichment: Additionally Search for Email<br>**Use this workflow** to enrich leads with web search and **LeadIQ email** (if available). |
| LeadIQ Database (Find Email) | HTTP Request | Call LeadIQ GraphQL to fetch email/phones | GraphQL Query #2 | Update Record | ## Step #2: Enrichment: Additionally Search for Email<br>**Use this workflow** to enrich leads with web search and **LeadIQ email** (if available). |
| Update Record | Airtable | Update Airtable Contact with enrichment | LeadIQ Database (Find Email) | — | ## Step #2: Enrichment: Additionally Search for Email<br>**Use this workflow** to enrich leads with web search and **LeadIQ email** (if available). |
| Sticky Note | Sticky Note | Comment | — | — | ## Manage Number of Leads<br>Set the number of LeadIQ credits you want to spend per single run:<br>**Change the code line**:  <input.limit = 1><br>(enter your number instead) |
| Sticky Note1 | Sticky Note | Comment | — | — | ## Step #1: Lead Research & List Building<br>**Use this workflow** to build a list of leads with **LeadIQ** and save them to **Airtable**. Brown notes explain specific nodes and nuances. |
| Sticky Note2 | Sticky Note | Comment | — | — | ## Step #2: Enrichment: Additionally Search for Email<br>**Use this workflow** to enrich leads with web search and **LeadIQ email** (if available). |
| Sticky Note3 | Sticky Note | Comment | — | — | ## Search Filters Summary<br>This agent combines filters from the contact and company data agents to build a GraphQL query. |
| Sticky Note4 | Sticky Note | Comment | — | — | ## Web Search<br>This agent finds missing company data on the internet. |
| Sticky Note5 | Sticky Note | Comment | — | — | 🎥 [Video Trailer – How It Works](https://vimeo.com/1151100805)<br><br>## How it Works<br><br>1. Start from "On Chat Message" Trigger Node. Use a simple prompt to get leads, example: «Founder at software engineering firm, 11-50 people size, based in New York, using AI technologies». The workflow has two AI agents that will transform your input into closest existing fiters, so you can make the prompt in your own words;<br><br>2. After using Step #1 workflows and creating the contact, search for an email by using Step #2.<br><br>## Steps Description<br><br>###Step #1<br><br>1. Pay no attention to the Structured Output Parser error — it still works. The reason for the error is the GraphQL schema used by the provider.<br><br>2. FlatAdvancedSearch (FlatSearchInput) are GraphQL methods used to find a list of people in LeadIQ.<br><br>###Step #2<br><br>4. Make a separate column in the “contacts” table of your Airtable CRM and name it “campaign” — you need this to identify which contacts in your table to enrich.<br><br>5. In the Airtable “Search records…” node, use the following search by formula: {Campaign} = "<your campaign name here>"<br><br>6. SearchPeople (SearchPeopleInput) are GraphQL methods used to find emails. This action triggers spending GraphQL credits.<br><br>## Setup Steps<br><br>1. Sign up for Airtable, find a "Sales CRM" template (left panel → templates and apps → marketing → «sales CRM»)<br><br>2. Set the correct base/sheet in all Airtable nodes;<br><br>3. Sign up for LeadIQ (https://leadiq.com), get API string called «Secret Base64 API key»<br><br>4. Set it into all HTTP nodes, make a POST request to URL:<br>https://api.leadiq.com/graphql. Turn on «Send Headers» and place the following headers:<br>- Authorization : Basis <your API string here><br>- Content-Type : application/json;<br><br>5. Place the Mistral model according to each MistralAI node’s name. Experimenting with AI models outside of Mistral may break the output of the current agentic prompts. |
| Sticky Note7 | Sticky Note | Comment | — | — | ## Airtable Note<br>Account should be an array, even if it's a single record. To add a contact you have to obtaine ID of it's account in array format |
| Sticky Note8 | Sticky Note | Comment | — | — | ## Start Here<br>Start the workflow by entering a prompt into the chat - describe which leads you want to find. |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow**
- Name: “Build a Prospecting List with LeadIQ and Save It to Airtable CRM”.

2) **Add Chat Trigger**
- Node: **When chat message received** (`Chat Trigger`)
- Set **Public = true**
- Title: “LeadIQ Lead Search Agent”
- Input placeholder: example lead prompt
- Initial message: greeting text

3) **Add Contact filter extraction chain**
- Add **Mistral** node: **Mistral Small Latest #1** (`mistral-small-latest`)
  - Configure **Mistral Cloud API credentials**
- Add **Chain LLM** node: **Contact-Level Search Criteria**
  - Prompt: strict JSON extraction for `jobTitles`, `role`, `seniority`, `locations`
  - Reference the chat input with `{{$json.chatInput}}`
- Connect:
  - Chat Trigger → Contact-Level Search Criteria
  - Mistral Small Latest #1 (ai_languageModel) → Contact-Level Search Criteria

4) **Add Company filter extraction chain**
- Add **Mistral** node: **Mistral Small Latest #2** (`mistral-small-latest`)
- Add **Chain LLM** node: **Company-level data preparation**
  - Prompt: strict JSON generator for LeadIQ `companyFilter` fields, using enumerated lists.
  - Use expression `{{ $('When chat message received').item.json.chatInput }}`
- Connect:
  - Contact-Level Search Criteria → Company-level data preparation
  - Mistral Small Latest #2 (ai_languageModel) → Company-level data preparation

5) **Add Filters Summary agent with structured output parsing**
- Add **Mistral** node: **Mistral Small Latest #3** (`mistral-small-latest`)
- Add **Agent** node: **Filters Summary Agent**
  - System message: “You are a LeadIQ search filter builder.”
  - Prompt merges:
    - `{{ $('Contact-Level Search Criteria').item.json.text }}`
    - `{{ $json.text }}` (company extraction output)
  - Enable “Has Output Parser” = true
- Add **Structured Output Parser** node
  - Enable `autoFix`
  - Provide a schema example matching your chosen final keys.
- Connect:
  - Company-level data preparation → Filters Summary Agent
  - Mistral Small Latest #3 (ai_languageModel) → Filters Summary Agent
  - Structured Output Parser (ai_outputParser) → Filters Summary Agent
  - (If you keep the workflow design) connect an LLM to the parser (as in JSON): **Open Mistral 7B** → Structured Output Parser
- **Important alignment step:** Ensure the agent output keys match what your GraphQL builder code expects (either singular `contactFilter/companyFilter` or plural `contactFilters/companyFilters`). Adjust one side so they match.

6) **Add LeadIQ “Find People” GraphQL builder**
- Add **Code** node: **GraphQL Query #1**
  - Paste logic that:
    - Reads merged filters from Filters Summary Agent output
    - Builds `FlatAdvancedSearch` GraphQL payload with `variables.input`
    - Sets `input.limit` and `input.skip`
  - Set `input.limit` to desired per-run credit spend.
- Connect: Filters Summary Agent → GraphQL Query #1

7) **Add LeadIQ HTTP Request (Find People)**
- Add **HTTP Request** node: **LeadIQ Database (Find People)**
  - Method: POST
  - URL: `https://api.leadiq.com/graphql`
  - Body: JSON = incoming `$json`
  - Headers:
    - `Authorization: Basic <Secret Base64 API key>`
    - `Content-Type: application/json`
- Connect: GraphQL Query #1 → LeadIQ Database (Find People)

8) **Add Web Enrichment Agent (Tavily + Mistral)**
- Add **Mistral** node: **Mistral Medium Latest** (`mistral-medium-latest`)
- Add **Tavily Tool** node
  - Configure **Tavily API credentials**
- Add **Agent** node: **Web Enrichment Agent**
  - Provide strict JSON output instructions with keys:
    - `company_linkedin_url`, `company_description`, `company_hq_address`
  - Reference LeadIQ fields via expressions (company name, domain, etc.)
  - Set `onError` to “continue” if desired
- Connect:
  - LeadIQ Database (Find People) → Web Enrichment Agent
  - Mistral Medium Latest (ai_languageModel) → Web Enrichment Agent
  - Tavily Tool (ai_tool) → Web Enrichment Agent

9) **Parse enrichment JSON**
- Add **Code** node: **Output Divider (Three Parts)**
  - Strip code fences and `JSON.parse()` the output into clean JSON.
- Connect: Web Enrichment Agent → Output Divider (Three Parts)

10) **Airtable: Upsert Account**
- Add **Airtable** node: **Airtable: Create Account**
  - Configure **Airtable Personal Access Token** credentials
  - Select Base: your CRM base
  - Table: `Accounts`
  - Operation: **Upsert**
  - Matching column: `id`
  - Map fields (examples):
    - `id`: LeadIQ person/company id
    - `Name`: company name
    - `Size`, `Industry`, `Company website`
    - Enrichment fields: LinkedIn/Description/HQ address
- Connect: Output Divider → Airtable: Create Account

11) **Get Account record (for linked record id)**
- Add **Airtable** node: **Get Record**
  - Base: same
  - Table: Accounts
  - Record ID: `{{ $json.id }}` (Airtable record id from upsert output)
- Connect: Airtable: Create Account → Get Record

12) **Format Account link as array**
- Add **Code** node: **Return an Array for Account**
  - Output: `{ account_link: [ accountRecordId ] }`
- Connect: Get Record → Return an Array for Account

13) **Airtable: Upsert Contact**
- Add **Airtable** node: **Add Contact**
  - Base: same
  - Table: `Contacts`
  - Operation: **Upsert**
  - Matching column: choose a stable contact identifier (recommended: LeadIQ person id or LinkedIn URL)
  - Map:
    - `Name`, `Title`, `LinkedIn`
    - `Account` (linked record field): `{{ $json.account_link }}`
- Connect: Return an Array for Account → Add Contact

14) **(Optional) Build Step #2 enrichment block**
- Add **Airtable Search** node: **Search and Filter Records by Campaign**
  - Table: Contacts
  - Filter formula: `{Campaign} = "<campaign name>"`
- Add **Code** node: **GraphQL Query #2**
  - Build `searchPeople` query with `linkedinUrl` and `limit`
- Add **HTTP Request** node: **LeadIQ Database (Find Email)**
  - Same endpoint/headers as before; enable retryOnFail
- Add **Airtable Update** node: **Update Record**
  - Update the same contact record ID
  - Map email/phone safely (use conditional expressions or Code node to avoid out-of-range indexes)
- Connect:
  - Search → GraphQL Query #2 → LeadIQ Find Email → Update Record
- Add a trigger (manual/cron/webhook) if you want this block to run automatically.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| 🎥 Video Trailer – How It Works | https://vimeo.com/1151100805 |
| LeadIQ signup and API key (“Secret Base64 API key”) | https://leadiq.com |
| LeadIQ GraphQL endpoint to use in HTTP nodes | `https://api.leadiq.com/graphql` |
| Headers required for LeadIQ HTTP nodes | `Authorization: Basic <your API string>` and `Content-Type: application/json` |
| Step #1 note about Structured Output Parser “error” | “Pay no attention… it still works… GraphQL schema used by the provider.” |
| Step #2 requires a `Campaign` column in Airtable Contacts | Use it to filter which contacts to enrich. |
| Model choice caution | Using non-Mistral models may break the strict JSON formatting assumptions in the prompts. |

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.