Enrich company leads with Firecrawl, OpenRouter AI, and Supabase

https://n8nworkflows.xyz/workflows/enrich-company-leads-with-firecrawl--openrouter-ai--and-supabase-13910


# Enrich company leads with Firecrawl, OpenRouter AI, and Supabase

# 1. Workflow Overview

This workflow receives a company URL through a webhook, validates and normalizes it, enriches the company profile using Firecrawl plus an OpenRouter-powered AI agent, checks Supabase for an existing record, stores the result if it is new, and finally returns the enriched profile to the caller.

Typical use cases:
- Lead enrichment for outbound sales or CRM intake
- Company intelligence collection from a website and related web context
- Structured extraction of signals such as industry, pricing, hiring, funding, and tech stack
- API-style enrichment endpoint that can be called from external systems

## 1.1 Input Reception and Validation

The workflow starts with an HTTP POST webhook. It expects a JSON body containing a `url` field. A Code node sanitizes the input, strips protocol/path, validates the domain format, and normalizes it to an `https://` URL.

## 1.2 AI-Driven Enrichment

A LangChain agent receives the normalized URL. It uses:
- an OpenRouter chat model as the LLM,
- a Firecrawl scrape tool to inspect the company website,
- a Firecrawl search tool for supplementary context when needed,
- a structured output parser to force a predictable JSON schema.

This block is responsible for producing a normalized, machine-usable company profile.

## 1.3 Deduplication and Persistence

The structured profile is checked against Supabase using the domain as the key. If a record already exists, the workflow skips insertion. If not, it prepares the AI output for insertion and stores it in the `lead_enrichment` table.

## 1.4 API Response Handling

The workflow returns either:
- a URL validation error with HTTP 422, or
- the enriched profile with HTTP 200.

Because the webhook is configured in `responseNode` mode, the response is controlled explicitly by Respond to Webhook nodes.

---

# 2. Block-by-Block Analysis

## Block 1 — Input Reception and Validation

### Overview
This block receives the inbound POST request and validates the supplied company URL. It normalizes the input into a clean domain and canonical `https://` URL before downstream processing.

### Nodes Involved
- Receive company URL
- Validate and normalize URL
- Return URL validation error

### Node Details

#### 1. Receive company URL
- **Type and technical role:** `n8n-nodes-base.webhook`  
  Entry point for the workflow. Receives external HTTP requests.
- **Configuration choices:**
  - Method: `POST`
  - Path: generated webhook path `dedaa64a-3dc9-43ea-82ac-7fac034af0b2`
  - Response mode: `responseNode`
- **Key expressions or variables used:**
  - No major expressions in this node itself
- **Input and output connections:**
  - No input
  - Output → `Validate and normalize URL`
- **Version-specific requirements:**
  - Uses webhook node version `2.1`
  - `responseNode` mode requires at least one `Respond to Webhook` node downstream
- **Edge cases or potential failure types:**
  - Caller may send malformed JSON
  - Caller may omit `body.url`
  - If no response node is reached, the request can hang or fail
- **Sub-workflow reference:** None

#### 2. Validate and normalize URL
- **Type and technical role:** `n8n-nodes-base.code`  
  Performs input extraction, domain cleanup, validation, and normalization.
- **Configuration choices:**
  - JavaScript Code node
  - `onError: continueErrorOutput`, so runtime errors are routed to the error output rather than terminating the workflow
- **Key expressions or variables used:**
  - `const body = $input.first().json.body;`
  - `const raw = body?.url?.trim();`
  - Regex removes protocol and path
  - Domain validation regex:
    - `^[a-zA-Z0-9]([a-zA-Z0-9\-]{0,61}[a-zA-Z0-9])?(\.[a-zA-Z]{2,})+$`
- **Behavior:**
  - If `url` is missing, returns:
    - `status: 422`
    - message indicating missing field
  - If format is invalid, throws an error
  - If valid, returns:
    - `status: 200`
    - `domain`
    - normalized `url` as `https://<domain>`
- **Input and output connections:**
  - Input ← `Receive company URL`
  - Main success output → `Extract business signals with AI`
  - Error output → `Return URL validation error`
- **Version-specific requirements:**
  - Uses Code node version `2`
- **Edge cases or potential failure types:**
  - Non-string or oddly formatted `url`
  - URLs containing unsupported host patterns
  - Subdomains are accepted if they match the regex
  - Internationalized domain names may not validate unless pre-converted to punycode
  - Missing `body` structure may lead to fallback behavior
- **Sub-workflow reference:** None

#### 3. Return URL validation error
- **Type and technical role:** `n8n-nodes-base.respondToWebhook`  
  Sends an explicit HTTP error response back to the webhook caller.
- **Configuration choices:**
  - Response code: `422`
  - Response body key: `={{ $json.error }}`
- **Key expressions or variables used:**
  - `{{ $json.error }}`
- **Input and output connections:**
  - Input ← error output of `Validate and normalize URL`
  - No downstream output
- **Version-specific requirements:**
  - Uses Respond to Webhook version `1.5`
- **Edge cases or potential failure types:**
  - The error object shape may differ from the expected structure
  - If the Code node returns a normal item with `status: 422` rather than throwing, that item goes through the main path, not this error path
  - As configured, a missing `url` does not throw; it returns a normal object, so it may not reach this node
- **Sub-workflow reference:** None

**Important implementation note:**  
There is a logical inconsistency in this block. The Code node handles a missing `url` by returning a normal item rather than throwing an error, but the response node is connected to the error output only. As a result, missing `url` may continue into the AI block instead of returning 422. Only thrown errors, such as invalid URL format, are reliably routed to the validation error response.

---

## Block 2 — AI-Driven Enrichment

### Overview
This block uses an AI agent to inspect the company website and optionally search for supporting information. It converts the findings into a structured profile with a stable schema.

### Nodes Involved
- Extract business signals with AI
- OpenRouter LLM
- Scrape company website
- Search company data
- Parse into structured profile

### Node Details

#### 4. Extract business signals with AI
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent`  
  Central AI agent orchestrating the LLM, tools, and structured output parser.
- **Configuration choices:**
  - Prompt type: defined directly in the node
  - Input text: `url: {{ $json.url }}`
  - System message instructs the agent to:
    - use Firecrawl tools,
    - scrape first,
    - use search only if additional context is needed,
    - return structured business signals
  - Output parser enabled
- **Key expressions or variables used:**
  - `=url: {{ $json.url }}`
- **Expected output structure:**
  - `domain`
  - `company_name`
  - `industry`
  - `pricing_model`
  - `has_free_trial`
  - `employee_signal`
  - `funding_stage`
  - `tech_stack`
  - `integrations`
  - `target_customer`
  - `trust_signals`
  - `hiring`
  - `open_roles_count`
- **Input and output connections:**
  - Main input ← `Validate and normalize URL`
  - LLM input ← `OpenRouter LLM`
  - AI tool inputs ← `Scrape company website`, `Search company data`
  - Output parser input ← `Parse into structured profile`
  - Main output → `Check for duplicate in Supabase`
- **Version-specific requirements:**
  - Uses LangChain Agent version `3.1`
  - Requires compatible AI nodes in the same workflow
- **Edge cases or potential failure types:**
  - LLM may produce incomplete or ambiguous data
  - Tool calls may fail due to API rate limits, auth issues, or target site blocking
  - If parser schema cannot be satisfied, execution may fail
  - Poorly structured websites may reduce extraction quality
- **Sub-workflow reference:** None

#### 5. OpenRouter LLM
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenRouter`  
  Provides the chat model used by the AI agent.
- **Configuration choices:**
  - Model: `anthropic/claude-sonnet-4.6`
  - No advanced options configured
- **Key expressions or variables used:**
  - None
- **Input and output connections:**
  - Connected as AI language model → `Extract business signals with AI`
- **Version-specific requirements:**
  - Uses OpenRouter Chat model node version `1`
  - Requires valid OpenRouter API credentials
- **Edge cases or potential failure types:**
  - Invalid API key
  - Model availability changes
  - Token or context limits
  - Provider-side latency or outages
- **Sub-workflow reference:** None

#### 6. Scrape company website
- **Type and technical role:** `@mendable/n8n-nodes-firecrawl.firecrawlTool`  
  AI tool node for scraping the provided company site.
- **Configuration choices:**
  - Operation: `scrape`
  - `url` and `parsers` are delegated to the agent using `$fromAI(...)`
- **Key expressions or variables used:**
  - `={{ $fromAI('URL', '', 'string') }}`
  - `={{ $fromAI('Parsers', '', 'string') }}`
- **Input and output connections:**
  - Connected as AI tool → `Extract business signals with AI`
- **Version-specific requirements:**
  - Uses Firecrawl tool version `1`
  - Requires Firecrawl API credentials
- **Edge cases or potential failure types:**
  - Blocked scraping target
  - Timeout on large or JS-heavy sites
  - Firecrawl quota exhaustion
  - Agent may pass invalid or incomplete tool arguments
- **Sub-workflow reference:** None

#### 7. Search company data
- **Type and technical role:** `@mendable/n8n-nodes-firecrawl.firecrawlTool`  
  AI tool node used for supplementary web search.
- **Configuration choices:**
  - Resource: `MapSearch`
  - Operation: `search`
  - Result limit: `3`
  - `query` and `sources` are delegated to the agent using `$fromAI(...)`
- **Key expressions or variables used:**
  - `={{ $fromAI('Query', '', 'string') }}`
  - `={{ $fromAI('Sources', '', 'string') }}`
- **Input and output connections:**
  - Connected as AI tool → `Extract business signals with AI`
- **Version-specific requirements:**
  - Uses Firecrawl tool version `1`
  - Requires Firecrawl API credentials
- **Edge cases or potential failure types:**
  - Search results may be sparse or noisy
  - Rate limits or auth failures
  - Agent could formulate an ineffective query
- **Sub-workflow reference:** None

#### 8. Parse into structured profile
- **Type and technical role:** `@n8n/n8n-nodes-langchain.outputParserStructured`  
  Constrains the AI output into a predictable JSON object.
- **Configuration choices:**
  - Uses a JSON schema example rather than a formal JSON Schema definition
  - Example includes strings, booleans, integers, and arrays
- **Key expressions or variables used:**
  - No expressions; schema example is static
- **Input and output connections:**
  - Connected as AI output parser → `Extract business signals with AI`
- **Version-specific requirements:**
  - Uses version `1.3`
- **Edge cases or potential failure types:**
  - Missing fields or wrong types from the LLM
  - Arrays returned as strings
  - Numeric values such as `open_roles_count` may be estimated or omitted
- **Sub-workflow reference:** None

---

## Block 3 — Deduplication and Persistence

### Overview
This block checks whether the domain already exists in Supabase. Existing records bypass insert; new records are prepared from the AI output and inserted into the `lead_enrichment` table.

### Nodes Involved
- Check for duplicate in Supabase
- Skip if already enriched
- Prepare profile for insert
- Save enriched profile to Supabase

### Node Details

#### 9. Check for duplicate in Supabase
- **Type and technical role:** `n8n-nodes-base.supabase`  
  Queries Supabase for an existing record matching the extracted domain.
- **Configuration choices:**
  - Operation: `get`
  - Table: `lead_enrichment`
  - Filter condition:
    - `domain = {{ $json.output.domain }}`
  - `alwaysOutputData: true`
- **Key expressions or variables used:**
  - `={{ $json.output.domain }}`
- **Input and output connections:**
  - Input ← `Extract business signals with AI`
  - Output → `Skip if already enriched`
- **Version-specific requirements:**
  - Uses Supabase node version `1`
  - Requires Supabase credentials
- **Edge cases or potential failure types:**
  - Wrong table or schema setup
  - Missing `output.domain`
  - Supabase auth failures
  - If `alwaysOutputData` is enabled, downstream logic must account for possibly empty result items
- **Sub-workflow reference:** None

#### 10. Skip if already enriched
- **Type and technical role:** `n8n-nodes-base.if`  
  Determines whether a duplicate was found.
- **Configuration choices:**
  - Condition checks whether `{{$input.all()[0].json}}` is not empty
  - If true: treat as duplicate found
  - If false: proceed to insert
- **Key expressions or variables used:**
  - `={{$input.all()[0].json}}`
- **Input and output connections:**
  - Input ← `Check for duplicate in Supabase`
  - True output → `Return enriched profile`
  - False output → `Prepare profile for insert`
- **Version-specific requirements:**
  - Uses If node version `2.3`
- **Edge cases or potential failure types:**
  - This logic depends on how the Supabase get operation returns empty vs non-empty results
  - `alwaysOutputData` can complicate duplicate detection
  - If Supabase returns an empty object rather than no item, the condition may evaluate unexpectedly
- **Sub-workflow reference:** None

#### 11. Prepare profile for insert
- **Type and technical role:** `n8n-nodes-base.code`  
  Rebuilds the insert payload directly from the AI node output.
- **Configuration choices:**
  - JavaScript code maps the AI output into flat JSON items
- **Key expressions or variables used:**
  - `$('Extract business signals with AI').all().map(item => ({ json: item.json.output }))`
- **Input and output connections:**
  - Input ← false branch of `Skip if already enriched`
  - Output → `Save enriched profile to Supabase`
- **Version-specific requirements:**
  - Uses Code node version `2`
- **Edge cases or potential failure types:**
  - If AI output structure changes and `item.json.output` is absent, insert payload becomes invalid
  - If arrays/booleans do not match database column types, insertion may fail
- **Sub-workflow reference:** None

#### 12. Save enriched profile to Supabase
- **Type and technical role:** `n8n-nodes-base.supabase`  
  Inserts the prepared profile into the database.
- **Configuration choices:**
  - Table: `lead_enrichment`
  - Data mode: `autoMapInputData`
- **Key expressions or variables used:**
  - No custom expressions; input is mapped automatically
- **Input and output connections:**
  - Input ← `Prepare profile for insert`
  - Output → `Return enriched profile`
- **Version-specific requirements:**
  - Uses Supabase node version `1`
  - Requires Supabase credentials
- **Edge cases or potential failure types:**
  - Table schema mismatch
  - Nullability constraints not satisfied
  - Duplicate key conflict if uniqueness exists and duplicate detection missed
  - Auth or row-level security issues depending on credentials
- **Sub-workflow reference:** None

**Important implementation note:**  
Two different Supabase credentials are used:
- `Supabase account 2` for duplicate checking
- `Supabase account` for insert

This is valid, but it can create environment drift if they do not point to the same Supabase project or schema.

---

## Block 4 — API Response Handling

### Overview
This block returns the final API response to the caller. Both duplicate and newly inserted records lead to the same success response, while validation errors should return a 422 response.

### Nodes Involved
- Return enriched profile
- Return URL validation error

### Node Details

#### 13. Return enriched profile
- **Type and technical role:** `n8n-nodes-base.respondToWebhook`  
  Sends the enriched profile back to the API caller.
- **Configuration choices:**
  - Response code: `200`
  - Response body uses data from the AI node directly rather than the DB record
- **Key expressions or variables used:**
  - `={{ $('Extract business signals with AI').item.json }}`
- **Input and output connections:**
  - Input ← true branch of `Skip if already enriched`
  - Input ← `Save enriched profile to Supabase`
- **Version-specific requirements:**
  - Uses Respond to Webhook version `1.5`
- **Edge cases or potential failure types:**
  - Because the response uses the AI node output, it does not confirm what was actually stored in Supabase
  - If AI node output is unavailable in the current execution context, expression resolution may fail
- **Sub-workflow reference:** None

#### 14. Return URL validation error
- Already documented in Block 1 because it belongs functionally to input validation and HTTP response handling.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note | n8n-nodes-base.stickyNote | Documentation note for database setup |  |  | ## Supabase setup<br>Run the SQL migration from the workflow README to create the `lead_enrichment` table with auto-updating timestamps. |
| Sticky Note1 | n8n-nodes-base.stickyNote | Documentation note for workflow behavior and setup |  |  | ### How it works<br>1. A webhook receives a company URL via POST request<br>2. The URL is validated and normalized<br>3. Firecrawl scrapes the website and searches for additional company data<br>4. An AI agent (OpenRouter) extracts structured business signals: industry, pricing, tech stack, funding stage, hiring status, and more<br>5. Supabase checks for duplicates, then saves the enriched profile<br>6. The webhook returns the enriched result or a validation error<br><br>### Setup<br>1. Create a Supabase project and run the SQL from the "Supabase setup" sticky to create the `lead_enrichment` table<br>2. Add your Firecrawl API key<br>3. Add your OpenRouter API key (or swap for any OpenAI-compatible provider)<br>4. Add your Supabase credentials (URL + service role key)<br>5. Activate the workflow and send a POST request with `{"url": "firecrawl.dev"}` to the webhook |
| Receive company URL | n8n-nodes-base.webhook | Receives POST requests containing a company URL |  | Validate and normalize URL | ### How it works<br>1. A webhook receives a company URL via POST request<br>2. The URL is validated and normalized<br>3. Firecrawl scrapes the website and searches for additional company data<br>4. An AI agent (OpenRouter) extracts structured business signals: industry, pricing, tech stack, funding stage, hiring status, and more<br>5. Supabase checks for duplicates, then saves the enriched profile<br>6. The webhook returns the enriched result or a validation error<br><br>### Setup<br>1. Create a Supabase project and run the SQL from the "Supabase setup" sticky to create the `lead_enrichment` table<br>2. Add your Firecrawl API key<br>3. Add your OpenRouter API key (or swap for any OpenAI-compatible provider)<br>4. Add your Supabase credentials (URL + service role key)<br>5. Activate the workflow and send a POST request with `{"url": "firecrawl.dev"}` to the webhook |
| Validate and normalize URL | n8n-nodes-base.code | Validates request body input and normalizes the URL/domain | Receive company URL | Extract business signals with AI; Return URL validation error | ### How it works<br>1. A webhook receives a company URL via POST request<br>2. The URL is validated and normalized<br>3. Firecrawl scrapes the website and searches for additional company data<br>4. An AI agent (OpenRouter) extracts structured business signals: industry, pricing, tech stack, funding stage, hiring status, and more<br>5. Supabase checks for duplicates, then saves the enriched profile<br>6. The webhook returns the enriched result or a validation error<br><br>### Setup<br>1. Create a Supabase project and run the SQL from the "Supabase setup" sticky to create the `lead_enrichment` table<br>2. Add your Firecrawl API key<br>3. Add your OpenRouter API key (or swap for any OpenAI-compatible provider)<br>4. Add your Supabase credentials (URL + service role key)<br>5. Activate the workflow and send a POST request with `{"url": "firecrawl.dev"}` to the webhook |
| Extract business signals with AI | @n8n/n8n-nodes-langchain.agent | Orchestrates AI-driven extraction using tools and structured parsing | Validate and normalize URL; OpenRouter LLM; Scrape company website; Search company data; Parse into structured profile | Check for duplicate in Supabase |  |
| OpenRouter LLM | @n8n/n8n-nodes-langchain.lmChatOpenRouter | Supplies the chat language model for the AI agent |  | Extract business signals with AI |  |
| Scrape company website | @mendable/n8n-nodes-firecrawl.firecrawlTool | Firecrawl tool for scraping the company website |  | Extract business signals with AI |  |
| Search company data | @mendable/n8n-nodes-firecrawl.firecrawlTool | Firecrawl tool for supplementary company search |  | Extract business signals with AI |  |
| Parse into structured profile | @n8n/n8n-nodes-langchain.outputParserStructured | Forces the AI result into a structured JSON profile |  | Extract business signals with AI |  |
| Check for duplicate in Supabase | n8n-nodes-base.supabase | Looks up an existing company profile by domain | Extract business signals with AI | Skip if already enriched | ## Supabase setup<br>Run the SQL migration from the workflow README to create the `lead_enrichment` table with auto-updating timestamps. |
| Skip if already enriched | n8n-nodes-base.if | Branches based on whether a duplicate record exists | Check for duplicate in Supabase | Return enriched profile; Prepare profile for insert | ## Supabase setup<br>Run the SQL migration from the workflow README to create the `lead_enrichment` table with auto-updating timestamps. |
| Prepare profile for insert | n8n-nodes-base.code | Builds a database-ready payload from the AI output | Skip if already enriched | Save enriched profile to Supabase | ## Supabase setup<br>Run the SQL migration from the workflow README to create the `lead_enrichment` table with auto-updating timestamps. |
| Save enriched profile to Supabase | n8n-nodes-base.supabase | Inserts the enriched profile into Supabase | Prepare profile for insert | Return enriched profile | ## Supabase setup<br>Run the SQL migration from the workflow README to create the `lead_enrichment` table with auto-updating timestamps. |
| Return enriched profile | n8n-nodes-base.respondToWebhook | Returns the enriched JSON profile with HTTP 200 | Skip if already enriched; Save enriched profile to Supabase |  |  |
| Return URL validation error | n8n-nodes-base.respondToWebhook | Returns a validation error with HTTP 422 | Validate and normalize URL |  |  |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it something like: `Enrich leads with Firecrawl and store in Supabase`.

2. **Add a Webhook node**
   - Node type: `Webhook`
   - Name: `Receive company URL`
   - HTTP Method: `POST`
   - Path: choose a fixed path or generate one
   - Response Mode: `Using Respond to Webhook Node` / `responseNode`

3. **Add a Code node for validation**
   - Node type: `Code`
   - Name: `Validate and normalize URL`
   - Enable error continuation to error output (`continueErrorOutput`)
   - Paste logic equivalent to:
     - Read `body.url`
     - Trim it
     - If missing, return a 422-style payload
     - Strip protocol and path
     - Validate domain with regex
     - Throw on invalid format
     - Return normalized `domain` and `https://<domain>` URL
   - Connect:
     - `Receive company URL` → `Validate and normalize URL`

4. **Add a Respond to Webhook node for invalid input**
   - Node type: `Respond to Webhook`
   - Name: `Return URL validation error`
   - Response Code: `422`
   - Response Body / Response Key: expression using the error object, for example `{{ $json.error }}`
   - Connect:
     - Error output of `Validate and normalize URL` → `Return URL validation error`

5. **Add an AI Agent node**
   - Node type: `AI Agent` (LangChain agent)
   - Name: `Extract business signals with AI`
   - Prompt Type: `Define below`
   - Text input:
     - `url: {{ $json.url }}`
   - System message:
     - Instruct the model to use Firecrawl tools to extract business signals
     - Prefer scrape first
     - Use search only if needed
     - Return structured company data including domain, company name, industry, pricing, trial, employee size signal, funding, stack, integrations, target customer, trust signals, hiring, and open roles count
   - Enable structured output parser
   - Connect:
     - Main output of `Validate and normalize URL` → `Extract business signals with AI`

6. **Add the OpenRouter chat model**
   - Node type: `OpenRouter Chat Model`
   - Name: `OpenRouter LLM`
   - Model: `anthropic/claude-sonnet-4.6`
   - Credentials:
     - Create OpenRouter credentials with your API key
   - Connect as AI language model to:
     - `Extract business signals with AI`

7. **Add a Firecrawl tool for scraping**
   - Node type: `Firecrawl Tool`
   - Name: `Scrape company website`
   - Operation: `scrape`
   - For tool parameters, allow the AI to supply them:
     - URL via `$fromAI('URL', '', 'string')`
     - Parsers via `$fromAI('Parsers', '', 'string')`
   - Credentials:
     - Create Firecrawl API credentials
   - Connect as AI tool to:
     - `Extract business signals with AI`

8. **Add a Firecrawl tool for search**
   - Node type: `Firecrawl Tool`
   - Name: `Search company data`
   - Resource: `MapSearch`
   - Operation: `search`
   - Limit: `3`
   - Parameters from AI:
     - Query via `$fromAI('Query', '', 'string')`
     - Sources via `$fromAI('Sources', '', 'string')`
   - Credentials:
     - Reuse the same Firecrawl credentials
   - Connect as AI tool to:
     - `Extract business signals with AI`

9. **Add a Structured Output Parser**
   - Node type: `Structured Output Parser`
   - Name: `Parse into structured profile`
   - Provide a JSON schema example with fields:
     - `domain`
     - `company_name`
     - `industry`
     - `pricing_model`
     - `has_free_trial`
     - `employee_signal`
     - `funding_stage`
     - `tech_stack`
     - `integrations`
     - `target_customer`
     - `trust_signals`
     - `hiring`
     - `open_roles_count`
   - Connect as AI output parser to:
     - `Extract business signals with AI`

10. **Create the Supabase table first**
    - In Supabase, create a table named `lead_enrichment`
    - Include columns matching the AI output fields
    - Include auto-managed timestamps such as `created_at` and `updated_at`
    - If you want reliable deduplication, make `domain` unique
    - Recommended credentials for n8n:
      - Supabase URL
      - Service role key, especially if row-level security would block writes

11. **Add a Supabase node for duplicate lookup**
    - Node type: `Supabase`
    - Name: `Check for duplicate in Supabase`
    - Operation: `Get`
    - Table: `lead_enrichment`
    - Filter:
      - `domain` equals `{{ $json.output.domain }}`
    - Enable `Always Output Data`
    - Credentials:
      - Add Supabase credentials
    - Connect:
      - `Extract business signals with AI` → `Check for duplicate in Supabase`

12. **Add an If node for deduplication**
    - Node type: `If`
    - Name: `Skip if already enriched`
    - Configure a condition that checks whether the Supabase lookup returned a non-empty object/result
    - In the original workflow, the expression is:
      - `{{ $input.all()[0].json }} is not empty`
    - Connect:
      - `Check for duplicate in Supabase` → `Skip if already enriched`

13. **Add a Code node to prepare insert payload**
    - Node type: `Code`
    - Name: `Prepare profile for insert`
    - Use code equivalent to:
      - Read the output from `Extract business signals with AI`
      - Return `item.json.output` as the item JSON for insertion
    - Connect:
      - False branch of `Skip if already enriched` → `Prepare profile for insert`

14. **Add a Supabase insert node**
    - Node type: `Supabase`
    - Name: `Save enriched profile to Supabase`
    - Table: `lead_enrichment`
    - Data to send: `Auto-map input data`
    - Credentials:
      - Use Supabase credentials
    - Connect:
      - `Prepare profile for insert` → `Save enriched profile to Supabase`

15. **Add the success response node**
    - Node type: `Respond to Webhook`
    - Name: `Return enriched profile`
    - Response Code: `200`
    - Response body:
      - Use the AI result directly, for example `{{ $('Extract business signals with AI').item.json }}`
    - Connect:
      - True branch of `Skip if already enriched` → `Return enriched profile`
      - `Save enriched profile to Supabase` → `Return enriched profile`

16. **Optionally add sticky notes**
    - One note for setup instructions
    - One note for Supabase table setup
    - These are documentation-only and do not affect execution

17. **Configure credentials**
    - **OpenRouter**
      - API key required
      - Ensure selected model is available to your account
    - **Firecrawl**
      - API key required
      - Confirm your plan supports scrape and search
    - **Supabase**
      - Project URL
      - Service role key recommended for server-side inserts and reads

18. **Test the webhook**
    - Activate or run in test mode
    - Send a POST request such as:
      - `{"url":"firecrawl.dev"}`
    - Confirm:
      - valid URLs produce structured enrichment
      - invalid URLs produce 422 error
      - repeated calls skip duplicate insertion

19. **Recommended hardening changes**
    - Fix validation so missing `url` also goes to the error response path
   - Normalize case for domains, e.g. lowercase before querying/storing
    - Add a unique constraint on `domain`
    - Return the database record if you want confirmation of persisted data
    - Ensure both Supabase nodes use the same credential set unless separation is intentional

### Sub-workflow setup
This workflow does **not** use any Execute Workflow or sub-workflow nodes. No sub-workflow configuration is required.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Supabase setup: run the SQL migration from the workflow documentation to create the `lead_enrichment` table with auto-updating timestamps. | Database preparation |
| The workflow expects Firecrawl, OpenRouter, and Supabase credentials before activation. | Initial platform setup |
| Example request body: `{"url": "firecrawl.dev"}` | Webhook testing |
| OpenRouter can be replaced by another OpenAI-compatible provider if desired. | LLM provider flexibility |

