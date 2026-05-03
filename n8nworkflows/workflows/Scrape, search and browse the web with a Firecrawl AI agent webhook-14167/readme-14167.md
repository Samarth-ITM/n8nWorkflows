Scrape, search and browse the web with a Firecrawl AI agent webhook

https://n8nworkflows.xyz/workflows/scrape--search-and-browse-the-web-with-a-firecrawl-ai-agent-webhook-14167


# Scrape, search and browse the web with a Firecrawl AI agent webhook

# 1. Workflow Overview

This workflow exposes a single POST webhook that accepts a natural-language web research request and an optional JSON output schema. An AI agent then uses Firecrawl tools to search, scrape, and, when necessary, browse JavaScript-heavy sites before returning structured JSON to the caller.

Its main use cases are web data extraction scenarios where the user does not want to hardcode scraping logic, such as lead enrichment, market research, competitor monitoring, content collection, and ad hoc structured extraction from websites.

## 1.1 Input Reception and Schema Validation

The workflow begins with a webhook endpoint that receives the request payload. It validates whether the optional `output_schema` is a usable JSON Schema-like object and either passes a normalized schema downstream or returns an immediate error response.

## 1.2 AI Agent Orchestration

The central AI agent receives the user prompt, runs with a primary and fallback language model, and is instructed to use Firecrawl strategically. It decides whether to scrape directly, search first, or escalate to browser automation if the target site is JavaScript-heavy or requires interaction.

## 1.3 Firecrawl Tooling Layer

The agent has access to several Firecrawl tools:
- webpage scraping
- web search
- browser context guidance
- browser session creation
- browser code execution
- browser session listing
- browser session deletion

These tools are not chained in the normal execution graph; instead, they are exposed to the agent as callable tools.

## 1.4 Structured Output Formatting and Webhook Response

The agent’s result is constrained through a structured output parser configured with the validated schema. The final structured JSON is then returned through the webhook response node.

---

# 2. Block-by-Block Analysis

## 2.1 Input Reception and Validation

### Overview
This block receives incoming webhook calls and ensures the optional schema is usable before invoking the AI agent. If the schema is malformed, the workflow returns a clear error immediately rather than letting the agent fail later.

### Nodes Involved
- Receive Scrape Request
- Validate Output Schema
- Return Schema Error

### Node Details

#### Receive Scrape Request
- **Type and technical role:** `n8n-nodes-base.webhook`; entry point for external HTTP clients.
- **Configuration choices:**
  - HTTP method is `POST`
  - Path is `scrape-agent`
  - Response mode is `responseNode`, meaning another node will send the HTTP response
- **Key expressions or variables used:** None in node config; downstream nodes read `body.prompt` and `body.output_schema`
- **Input and output connections:**
  - No input; this is a trigger
  - Outputs to `Validate Output Schema`
- **Version-specific requirements:** Uses webhook node type version `2.1`
- **Edge cases or potential failure types:**
  - Caller sends invalid JSON body
  - Missing `body.prompt`
  - Webhook path conflicts with another active workflow
  - If the workflow is inactive, production webhook will not run
- **Sub-workflow reference:** None

#### Validate Output Schema
- **Type and technical role:** `n8n-nodes-base.code`; validates and normalizes the user-provided output schema.
- **Configuration choices:**
  - Reads the incoming request body from the first input item
  - If no `output_schema` is provided, it injects a permissive default:
    - `type: "object"`
    - `additionalProperties: true`
  - If `output_schema` exists, it checks:
    - it must be an object
    - it must not be an array
    - it must contain a `type`
    - `type` must be one of `object`, `array`, `string`, `number`, `boolean`
  - `onError` is set to `continueErrorOutput`, so schema-validation failures can be routed to an alternate branch
- **Key expressions or variables used:**
  - `const body = $input.first().json.body;`
  - outputs:
    - `json.prompt`
    - `json.output_schema`
- **Input and output connections:**
  - Input from `Receive Scrape Request`
  - Main success output to `Research & Extract Web Data`
  - Error output path to `Return Schema Error` via n8n’s continue-on-error behavior
- **Version-specific requirements:** Uses code node type version `2`
- **Edge cases or potential failure types:**
  - `body` missing entirely
  - `output_schema` sent as a string instead of parsed JSON object
  - unsupported `type` values like `integer`, `null`, or nested schema issues not explicitly validated here
  - prompt omitted; node does not reject it explicitly, so downstream failures may occur
- **Sub-workflow reference:** None

#### Return Schema Error
- **Type and technical role:** `n8n-nodes-base.respondToWebhook`; returns a static JSON error when schema validation fails.
- **Configuration choices:**
  - Responds with JSON
  - Includes:
    - `error: true`
    - explanatory message
    - example schema
- **Key expressions or variables used:** None; static JSON body
- **Input and output connections:**
  - Input from `Validate Output Schema` error branch
  - No output
- **Version-specific requirements:** Uses respond node version `1.5`
- **Edge cases or potential failure types:**
  - If webhook response has already been sent elsewhere, this would fail, though that should not happen in this design
  - Error message is generic and may not expose the exact schema issue
- **Sub-workflow reference:** None

---

## 2.2 AI Agent Orchestration

### Overview
This block contains the main decision-making logic. The agent receives the user’s prompt, uses a primary LLM with a fallback model, and is instructed to choose between search, scrape, and browser automation depending on how explicit and accessible the target information is.

### Nodes Involved
- Research & Extract Web Data
- Primary Chat Model
- Fallback Chat Model

### Node Details

#### Research & Extract Web Data
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent`; central autonomous agent coordinating tool usage and output generation.
- **Configuration choices:**
  - Prompt source is defined explicitly from the webhook body:
    - `$('Receive Scrape Request').item.json.body.prompt`
  - `maxIterations` is `40`, allowing fairly complex tool-based reasoning
  - Has a custom system message directing behavior:
    - use Firecrawl tools
    - search only when URL is not already known
    - scrape first if URL is given
    - use browser only for JS-heavy or interactive sites
    - use Browser Context before browser tools
    - do not answer in Markdown
  - `needsFallback` enabled
  - `hasOutputParser` enabled
- **Key expressions or variables used:**
  - `={{ $('Receive Scrape Request').item.json.body.prompt }}`
- **Input and output connections:**
  - Main input from `Validate Output Schema`
  - Main output to `Return Structured Results`
  - AI language model inputs from:
    - `Primary Chat Model`
    - `Fallback Chat Model`
  - AI output parser input from `Format Response to Schema`
  - AI tools from all Firecrawl tool nodes
- **Version-specific requirements:** Uses LangChain agent node version `3.1`
- **Edge cases or potential failure types:**
  - missing or empty prompt
  - model provider errors
  - agent exceeds iteration or token limits
  - tool call hallucinations or invalid tool arguments
  - schema too strict for the data that can be found
  - browser sessions left undeleted if the agent does not complete the full intended browser lifecycle
- **Sub-workflow reference:** None

#### Primary Chat Model
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenRouter`; main LLM provider for the agent.
- **Configuration choices:**
  - Model: `anthropic/claude-sonnet-4.6`
  - Minimal extra options configured
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Connects as `ai_languageModel` to `Research & Extract Web Data`
- **Version-specific requirements:** OpenRouter chat node version `1`
- **Edge cases or potential failure types:**
  - invalid OpenRouter credentials
  - model unavailable or renamed
  - provider-side throttling or quota errors
- **Sub-workflow reference:** None

#### Fallback Chat Model
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenRouter`; secondary LLM if the primary one fails.
- **Configuration choices:**
  - Model: `anthropic/claude-haiku-4.5`
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Connects as fallback `ai_languageModel` to `Research & Extract Web Data`
- **Version-specific requirements:** OpenRouter chat node version `1`
- **Edge cases or potential failure types:**
  - same as primary model
  - if both models fail, the agent fails entirely
- **Sub-workflow reference:** None

---

## 2.3 Firecrawl Search and Scrape Tools

### Overview
These tools let the agent perform direct page scraping or search the web for candidate URLs. This is the primary extraction path for most straightforward use cases.

### Nodes Involved
- Scrape Webpage Content
- Search the Web

### Node Details

#### Scrape Webpage Content
- **Type and technical role:** `@mendable/n8n-nodes-firecrawl.firecrawlTool`; agent-callable Firecrawl scrape tool.
- **Configuration choices:**
  - Operation: `scrape`
  - URL is supplied dynamically by the AI tool interface
  - Parsers are also AI-supplied
- **Key expressions or variables used:**
  - `URL` via `$fromAI('URL', '', 'string')`
  - `Parsers` via `$fromAI('Parsers', '', 'string')`
- **Input and output connections:**
  - Connected as `ai_tool` to `Research & Extract Web Data`
- **Version-specific requirements:** Firecrawl node version `1`
- **Edge cases or potential failure types:**
  - invalid or blocked URL
  - anti-bot defenses
  - parser mismatch or unsupported parser value
  - timeout or partial page retrieval
- **Sub-workflow reference:** None

#### Search the Web
- **Type and technical role:** `@mendable/n8n-nodes-firecrawl.firecrawlTool`; agent-callable web search tool.
- **Configuration choices:**
  - Resource: `MapSearch`
  - Operation: `search`
  - Limit fixed at `3`
  - Query and sources are filled by the AI
  - System prompt explicitly requires at least one search source
- **Key expressions or variables used:**
  - `Query` via `$fromAI('Query', '', 'string')`
  - `Sources` via `$fromAI('Sources', '', 'string')`
- **Input and output connections:**
  - Connected as `ai_tool` to `Research & Extract Web Data`
- **Version-specific requirements:** Firecrawl node version `1`
- **Edge cases or potential failure types:**
  - no sources specified by the agent
  - irrelevant search results due to vague prompts
  - API quota or network failure
- **Sub-workflow reference:** None

---

## 2.4 Firecrawl Browser Automation Tools

### Overview
This block supports advanced web interaction for sites that cannot be scraped reliably with simple HTTP fetching. The agent is instructed to consult browser context first, then manage Firecrawl browser sessions as needed.

### Nodes Involved
- Get Browser Context
- Create browser session in Firecrawl
- Execute browser code in Firecrawl
- List browser sessions in Firecrawl
- Delete browser session in Firecrawl

### Node Details

#### Get Browser Context
- **Type and technical role:** `@mendable/n8n-nodes-firecrawl.firecrawlTool`; informational tool that provides usage context for Firecrawl browser capabilities.
- **Configuration choices:**
  - Resource: `Browser`
  - `descriptionType` set to `manual`
  - Tool description explicitly says: “Browser context in Firecrawl. ALWAYS use this before using browser tools.”
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Connected as `ai_tool` to `Research & Extract Web Data`
- **Version-specific requirements:** Firecrawl node version `1`
- **Edge cases or potential failure types:**
  - limited practical value if the model ignores the instruction
  - credential or API availability issues
- **Sub-workflow reference:** None

#### Create browser session in Firecrawl
- **Type and technical role:** `@mendable/n8n-nodes-firecrawl.firecrawlTool`; starts a browser session for subsequent interactive actions.
- **Configuration choices:**
  - Resource: `Browser`
  - Operation: `browserCreate`
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Connected as `ai_tool` to `Research & Extract Web Data`
- **Version-specific requirements:** Firecrawl node version `1`
- **Edge cases or potential failure types:**
  - session creation failure
  - API quota exhaustion
  - agent creates a session but never closes it
- **Sub-workflow reference:** None

#### Execute browser code in Firecrawl
- **Type and technical role:** `@mendable/n8n-nodes-firecrawl.firecrawlTool`; runs browser automation code within a specific Firecrawl session.
- **Configuration choices:**
  - Resource: `Browser`
  - Operation: `browserExecute`
  - AI supplies:
    - execution code
    - timeout in seconds
    - session ID
- **Key expressions or variables used:**
  - `Code` via `$fromAI('Code', '', 'string')`
  - `Timeout__Seconds_` via `$fromAI('Timeout__Seconds_', '', 'number')`
  - `Session_ID` via `$fromAI('Session_ID', '', 'string')`
- **Input and output connections:**
  - Connected as `ai_tool` to `Research & Extract Web Data`
- **Version-specific requirements:** Firecrawl node version `1`
- **Edge cases or potential failure types:**
  - invalid or expired session ID
  - malformed browser code
  - timeout too short or too long
  - interactive flow not deterministic
- **Sub-workflow reference:** None

#### List browser sessions in Firecrawl
- **Type and technical role:** `@mendable/n8n-nodes-firecrawl.firecrawlTool`; enumerates active browser sessions.
- **Configuration choices:**
  - Resource: `Browser`
  - Operation: `browserList`
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Connected as `ai_tool` to `Research & Extract Web Data`
- **Version-specific requirements:** Firecrawl node version `1`
- **Edge cases or potential failure types:**
  - API errors
  - stale session information
- **Sub-workflow reference:** None

#### Delete browser session in Firecrawl
- **Type and technical role:** `@mendable/n8n-nodes-firecrawl.firecrawlTool`; closes a browser session to avoid lingering resources.
- **Configuration choices:**
  - Resource: `Browser`
  - Operation: `browserDelete`
  - Session ID supplied by the AI
- **Key expressions or variables used:**
  - `Session_ID` via `$fromAI('Session_ID', '', 'string')`
- **Input and output connections:**
  - Connected as `ai_tool` to `Research & Extract Web Data`
- **Version-specific requirements:** Firecrawl node version `1`
- **Edge cases or potential failure types:**
  - missing session ID
  - deleting already-closed sessions
  - orphaned sessions if not called
- **Sub-workflow reference:** None

---

## 2.5 Structured Output Formatting and Response

### Overview
This block forces the final answer into a structured JSON shape aligned with the validated schema. It then returns the parsed result to the original webhook caller.

### Nodes Involved
- Parser Chat Model
- Format Response to Schema
- Return Structured Results

### Node Details

#### Parser Chat Model
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenRouter`; dedicated LLM used by the structured output parser.
- **Configuration choices:**
  - Model: `anthropic/claude-haiku-4.5`
  - Separate from the agent’s main reasoning model
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Connected as `ai_languageModel` to `Format Response to Schema`
- **Version-specific requirements:** OpenRouter chat node version `1`
- **Edge cases or potential failure types:**
  - model unavailability
  - parser model may still struggle with very complex schemas
- **Sub-workflow reference:** None

#### Format Response to Schema
- **Type and technical role:** `@n8n/n8n-nodes-langchain.outputParserStructured`; converts the agent output into JSON matching the requested schema.
- **Configuration choices:**
  - `schemaType` is manual
  - `autoFix` is enabled, allowing the parser to attempt correction when the initial output does not fit perfectly
  - Schema is dynamically injected from the validated schema node
- **Key expressions or variables used:**
  - `={{ JSON.stringify($('Validate Output Schema').item.json.output_schema) }}`
- **Input and output connections:**
  - Receives `ai_languageModel` from `Parser Chat Model`
  - Connects as `ai_outputParser` to `Research & Extract Web Data`
- **Version-specific requirements:** Output parser node version `1.3`
- **Edge cases or potential failure types:**
  - schema references unsupported JSON Schema features
  - parser cannot coerce agent output into requested shape
  - auto-fix may produce approximate rather than exact semantic mapping
- **Sub-workflow reference:** None

#### Return Structured Results
- **Type and technical role:** `n8n-nodes-base.respondToWebhook`; sends the final result back to the HTTP caller.
- **Configuration choices:**
  - Uses default response behavior from incoming item data
- **Key expressions or variables used:** None explicitly
- **Input and output connections:**
  - Input from `Research & Extract Web Data`
- **Version-specific requirements:** Respond node version `1.5`
- **Edge cases or potential failure types:**
  - no data returned from agent
  - serialization issues if the parsed output is malformed
- **Sub-workflow reference:** None

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Receive Scrape Request | n8n-nodes-base.webhook | Accepts POST requests on the scrape-agent endpoint |  | Validate Output Schema | ## Firecrawl Scrape Agent |
| Receive Scrape Request | n8n-nodes-base.webhook | Accepts POST requests on the scrape-agent endpoint |  | Validate Output Schema | Send a POST request with a natural language prompt and an optional JSON schema. An AI agent uses Firecrawl to research the web and return clean, structured data matching your schema. |
| Receive Scrape Request | n8n-nodes-base.webhook | Accepts POST requests on the scrape-agent endpoint |  | Validate Output Schema | **Use cases:** data enrichment, lead generation, market research, competitor analysis, content aggregation. |
| Receive Scrape Request | n8n-nodes-base.webhook | Accepts POST requests on the scrape-agent endpoint |  | Validate Output Schema | ## How it works |
| Receive Scrape Request | n8n-nodes-base.webhook | Accepts POST requests on the scrape-agent endpoint |  | Validate Output Schema | 1. **Receive Scrape Request** accepts a POST with a `prompt` and optional `output_schema`. |
| Receive Scrape Request | n8n-nodes-base.webhook | Accepts POST requests on the scrape-agent endpoint |  | Validate Output Schema | 2. **Validate Output Schema** checks if the schema is valid JSON. If missing, a permissive default is used. If malformed, a clear error is returned immediately. |
| Receive Scrape Request | n8n-nodes-base.webhook | Accepts POST requests on the scrape-agent endpoint |  | Validate Output Schema | 3. **Research & Extract Web Data** takes the prompt and autonomously uses Firecrawl tools (scrape, search, browser automation) to find the requested information. |
| Receive Scrape Request | n8n-nodes-base.webhook | Accepts POST requests on the scrape-agent endpoint |  | Validate Output Schema | 4. **Format Response to Schema** parses the agent's raw output into structured JSON matching the provided or default schema. |
| Receive Scrape Request | n8n-nodes-base.webhook | Accepts POST requests on the scrape-agent endpoint |  | Validate Output Schema | 5. **Return Structured Results** sends the final JSON back to the caller. |
| Receive Scrape Request | n8n-nodes-base.webhook | Accepts POST requests on the scrape-agent endpoint |  | Validate Output Schema | ## Setup steps |
| Receive Scrape Request | n8n-nodes-base.webhook | Accepts POST requests on the scrape-agent endpoint |  | Validate Output Schema | 1. **Firecrawl API key:** Sign up at [firecrawl.dev](https://www.firecrawl.dev) and connect your key to all Firecrawl credential nodes. |
| Receive Scrape Request | n8n-nodes-base.webhook | Accepts POST requests on the scrape-agent endpoint |  | Validate Output Schema | 2. **LLM providers:** Configure the Primary Chat Model and Fallback Chat Model nodes with your preferred provider (OpenRouter, OpenAI, Anthropic, etc.). Also set up the Parser Chat Model used by the output formatter. |
| Receive Scrape Request | n8n-nodes-base.webhook | Accepts POST requests on the scrape-agent endpoint |  | Validate Output Schema | 3. **Activate the workflow** and copy your webhook URL. |
| Receive Scrape Request | n8n-nodes-base.webhook | Accepts POST requests on the scrape-agent endpoint |  | Validate Output Schema | 4. **Test it:** Send a simple POST with just a `prompt` field to confirm everything works. |
| Receive Scrape Request | n8n-nodes-base.webhook | Accepts POST requests on the scrape-agent endpoint |  | Validate Output Schema | ## Input & Validation |
| Validate Output Schema | n8n-nodes-base.code | Validates optional schema and normalizes request data | Receive Scrape Request | Research & Extract Web Data, Return Schema Error | ## Firecrawl Scrape Agent |
| Validate Output Schema | n8n-nodes-base.code | Validates optional schema and normalizes request data | Receive Scrape Request | Research & Extract Web Data, Return Schema Error | Send a POST request with a natural language prompt and an optional JSON schema. An AI agent uses Firecrawl to research the web and return clean, structured data matching your schema. |
| Validate Output Schema | n8n-nodes-base.code | Validates optional schema and normalizes request data | Receive Scrape Request | Research & Extract Web Data, Return Schema Error | **Use cases:** data enrichment, lead generation, market research, competitor analysis, content aggregation. |
| Validate Output Schema | n8n-nodes-base.code | Validates optional schema and normalizes request data | Receive Scrape Request | Research & Extract Web Data, Return Schema Error | ## How it works |
| Validate Output Schema | n8n-nodes-base.code | Validates optional schema and normalizes request data | Receive Scrape Request | Research & Extract Web Data, Return Schema Error | 1. **Receive Scrape Request** accepts a POST with a `prompt` and optional `output_schema`. |
| Validate Output Schema | n8n-nodes-base.code | Validates optional schema and normalizes request data | Receive Scrape Request | Research & Extract Web Data, Return Schema Error | 2. **Validate Output Schema** checks if the schema is valid JSON. If missing, a permissive default is used. If malformed, a clear error is returned immediately. |
| Validate Output Schema | n8n-nodes-base.code | Validates optional schema and normalizes request data | Receive Scrape Request | Research & Extract Web Data, Return Schema Error | 3. **Research & Extract Web Data** takes the prompt and autonomously uses Firecrawl tools (scrape, search, browser automation) to find the requested information. |
| Validate Output Schema | n8n-nodes-base.code | Validates optional schema and normalizes request data | Receive Scrape Request | Research & Extract Web Data, Return Schema Error | 4. **Format Response to Schema** parses the agent's raw output into structured JSON matching the provided or default schema. |
| Validate Output Schema | n8n-nodes-base.code | Validates optional schema and normalizes request data | Receive Scrape Request | Research & Extract Web Data, Return Schema Error | 5. **Return Structured Results** sends the final JSON back to the caller. |
| Validate Output Schema | n8n-nodes-base.code | Validates optional schema and normalizes request data | Receive Scrape Request | Research & Extract Web Data, Return Schema Error | ## Setup steps |
| Validate Output Schema | n8n-nodes-base.code | Validates optional schema and normalizes request data | Receive Scrape Request | Research & Extract Web Data, Return Schema Error | 1. **Firecrawl API key:** Sign up at [firecrawl.dev](https://www.firecrawl.dev) and connect your key to all Firecrawl credential nodes. |
| Validate Output Schema | n8n-nodes-base.code | Validates optional schema and normalizes request data | Receive Scrape Request | Research & Extract Web Data, Return Schema Error | 2. **LLM providers:** Configure the Primary Chat Model and Fallback Chat Model nodes with your preferred provider (OpenRouter, OpenAI, Anthropic, etc.). Also set up the Parser Chat Model used by the output formatter. |
| Validate Output Schema | n8n-nodes-base.code | Validates optional schema and normalizes request data | Receive Scrape Request | Research & Extract Web Data, Return Schema Error | 3. **Activate the workflow** and copy your webhook URL. |
| Validate Output Schema | n8n-nodes-base.code | Validates optional schema and normalizes request data | Receive Scrape Request | Research & Extract Web Data, Return Schema Error | 4. **Test it:** Send a simple POST with just a `prompt` field to confirm everything works. |
| Validate Output Schema | n8n-nodes-base.code | Validates optional schema and normalizes request data | Receive Scrape Request | Research & Extract Web Data, Return Schema Error | ## Input & Validation |
| Return Schema Error | n8n-nodes-base.respondToWebhook | Returns schema validation error JSON | Validate Output Schema |  | ## Input & Validation |
| Primary Chat Model | @n8n/n8n-nodes-langchain.lmChatOpenRouter | Primary reasoning model for the AI agent |  | Research & Extract Web Data | ## AI Agent & LLM Models |
| Fallback Chat Model | @n8n/n8n-nodes-langchain.lmChatOpenRouter | Backup reasoning model for the AI agent |  | Research & Extract Web Data | ## AI Agent & LLM Models |
| Research & Extract Web Data | @n8n/n8n-nodes-langchain.agent | Main AI agent that uses Firecrawl tools and schema parser | Validate Output Schema | Return Structured Results | ## AI Agent & LLM Models |
| Parser Chat Model | @n8n/n8n-nodes-langchain.lmChatOpenRouter | Model used by the structured output parser |  | Format Response to Schema | ## Output & Response |
| Format Response to Schema | @n8n/n8n-nodes-langchain.outputParserStructured | Enforces response schema on the AI output | Parser Chat Model | Research & Extract Web Data | ## Output & Response |
| Return Structured Results | n8n-nodes-base.respondToWebhook | Returns final structured JSON to the caller | Research & Extract Web Data |  | ## Output & Response |
| Scrape Webpage Content | @mendable/n8n-nodes-firecrawl.firecrawlTool | Firecrawl scrape tool callable by the agent |  | Research & Extract Web Data | ## Firecrawl Tools |
| Scrape Webpage Content | @mendable/n8n-nodes-firecrawl.firecrawlTool | Firecrawl scrape tool callable by the agent |  | Research & Extract Web Data | ## Search & Scrape |
| Search the Web | @mendable/n8n-nodes-firecrawl.firecrawlTool | Firecrawl search tool callable by the agent |  | Research & Extract Web Data | ## Firecrawl Tools |
| Search the Web | @mendable/n8n-nodes-firecrawl.firecrawlTool | Firecrawl search tool callable by the agent |  | Research & Extract Web Data | ## Search & Scrape |
| Get Browser Context | @mendable/n8n-nodes-firecrawl.firecrawlTool | Provides browser usage guidance to the agent |  | Research & Extract Web Data | ## Firecrawl Tools |
| Get Browser Context | @mendable/n8n-nodes-firecrawl.firecrawlTool | Provides browser usage guidance to the agent |  | Research & Extract Web Data | ## Browser Tool |
| Create browser session in Firecrawl | @mendable/n8n-nodes-firecrawl.firecrawlTool | Creates Firecrawl browser sessions |  | Research & Extract Web Data | ## Firecrawl Tools |
| Create browser session in Firecrawl | @mendable/n8n-nodes-firecrawl.firecrawlTool | Creates Firecrawl browser sessions |  | Research & Extract Web Data | ## Browser Tool |
| Execute browser code in Firecrawl | @mendable/n8n-nodes-firecrawl.firecrawlTool | Executes browser automation code in a Firecrawl session |  | Research & Extract Web Data | ## Firecrawl Tools |
| Execute browser code in Firecrawl | @mendable/n8n-nodes-firecrawl.firecrawlTool | Executes browser automation code in a Firecrawl session |  | Research & Extract Web Data | ## Browser Tool |
| List browser sessions in Firecrawl | @mendable/n8n-nodes-firecrawl.firecrawlTool | Lists active Firecrawl browser sessions |  | Research & Extract Web Data | ## Firecrawl Tools |
| List browser sessions in Firecrawl | @mendable/n8n-nodes-firecrawl.firecrawlTool | Lists active Firecrawl browser sessions |  | Research & Extract Web Data | ## Browser Tool |
| Delete browser session in Firecrawl | @mendable/n8n-nodes-firecrawl.firecrawlTool | Deletes Firecrawl browser sessions |  | Research & Extract Web Data | ## Firecrawl Tools |
| Delete browser session in Firecrawl | @mendable/n8n-nodes-firecrawl.firecrawlTool | Deletes Firecrawl browser sessions |  | Research & Extract Web Data | ## Browser Tool |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n and name it something like:
   - `Scrape, search and browse the web via a single AI agent webhook`

2. **Add a Webhook node** named `Receive Scrape Request`.
   - Set **HTTP Method** to `POST`
   - Set **Path** to `scrape-agent`
   - Set **Response Mode** to `Using Respond to Webhook Node` / `responseNode`
   - Leave other options at default unless you need authentication or CORS customization

3. **Add a Code node** named `Validate Output Schema`.
   - Connect `Receive Scrape Request` → `Validate Output Schema`
   - Set the node to run JavaScript
   - Enable behavior equivalent to **continue on error output** (`onError: continueErrorOutput`)
   - Paste this logic:
     - read `body` from the webhook payload
     - if `output_schema` is absent, return:
       - `prompt: body.prompt`
       - `output_schema: { type: "object", additionalProperties: true }`
     - if present, validate that:
       - it is an object
       - it is not an array
       - it has a `type`
       - `type` is one of `object`, `array`, `string`, `number`, `boolean`
     - return `prompt` and validated schema
   - The code in plain behavior form is:
     - reject malformed schema early
     - normalize missing schema to a permissive default

4. **Add a Respond to Webhook node** named `Return Schema Error`.
   - Connect it from the error branch of `Validate Output Schema`
   - Set **Respond With** to `JSON`
   - Set the response body to a static JSON object such as:
     - `error: true`
     - a message explaining valid schema types
     - an example schema showing an object with `name` and `price`

5. **Add an AI Agent node** named `Research & Extract Web Data`.
   - Connect `Validate Output Schema` main success output → `Research & Extract Web Data`
   - Set **Prompt Type** to `Define below`
   - Set the prompt expression to:
     - `$('Receive Scrape Request').item.json.body.prompt`
   - Enable **Fallback model**
   - Enable **Output parser**
   - Set **Max Iterations** to `40`
   - Use this system instruction logic:
     - use Firecrawl tools to search and scrape as needed
     - if URL is provided, scrape before searching
     - if site has heavy JS or needs clicking/pagination, use browser
     - use Browser Context before other browser tools
     - when using search, specify at least one source
     - do not return Markdown; use plain text structure only

6. **Add a chat model node** named `Primary Chat Model`.
   - Use `OpenRouter Chat Model` if available in your n8n instance
   - Set model to `anthropic/claude-sonnet-4.6`
   - Create or select **OpenRouter credentials**
   - Connect it to the AI Agent as the primary `ai_languageModel`

7. **Add another chat model node** named `Fallback Chat Model`.
   - Same node type as above
   - Set model to `anthropic/claude-haiku-4.5`
   - Use the same OpenRouter credential or another valid credential
   - Connect it to the AI Agent as the fallback `ai_languageModel`

8. **Add a structured output parser node** named `Format Response to Schema`.
   - Use the LangChain structured output parser node
   - Enable **Auto Fix**
   - Set **Schema Type** to `Manual`
   - Set the schema expression to:
     - `JSON.stringify($('Validate Output Schema').item.json.output_schema)`
   - Connect this parser to the AI Agent as `ai_outputParser`

9. **Add a chat model node** named `Parser Chat Model`.
   - Use the same OpenRouter chat model node type
   - Set model to `anthropic/claude-haiku-4.5`
   - Connect it to `Format Response to Schema` as its language model

10. **Add a Respond to Webhook node** named `Return Structured Results`.
    - Connect `Research & Extract Web Data` → `Return Structured Results`
    - Default response configuration is sufficient unless you want custom status codes or headers

11. **Add a Firecrawl tool node** named `Scrape Webpage Content`.
    - Node type: Firecrawl tool
    - Create or select **Firecrawl API credentials**
    - Set **Operation** to `scrape`
    - Set **URL** from AI input using the built-in AI parameter binding
    - Set **Parsers** from AI input as a string parameter
    - Connect it to `Research & Extract Web Data` as an `ai_tool`

12. **Add a Firecrawl tool node** named `Search the Web`.
    - Set **Resource** to `MapSearch`
    - Set **Operation** to `search`
    - Set **Limit** to `3`
    - Set **Query** from AI input
    - Set **Sources** from AI input
    - Connect it to the AI Agent as an `ai_tool`

13. **Add a Firecrawl tool node** named `Get Browser Context`.
    - Set **Resource** to `Browser`
    - Set a manual tool description:
      - `Browser context in Firecrawl. ALWAYS use this before using browser tools.`
    - Connect it to the AI Agent as an `ai_tool`

14. **Add a Firecrawl tool node** named `Create browser session in Firecrawl`.
    - Set **Resource** to `Browser`
    - Set **Operation** to `browserCreate`
    - Connect it to the AI Agent as an `ai_tool`

15. **Add a Firecrawl tool node** named `Execute browser code in Firecrawl`.
    - Set **Resource** to `Browser`
    - Set **Operation** to `browserExecute`
    - Map these parameters from AI input:
      - `Code` as string
      - `Timeout (Seconds)` as number
      - `Session ID` as string
    - Connect it to the AI Agent as an `ai_tool`

16. **Add a Firecrawl tool node** named `List browser sessions in Firecrawl`.
    - Set **Resource** to `Browser`
    - Set **Operation** to `browserList`
    - Connect it to the AI Agent as an `ai_tool`

17. **Add a Firecrawl tool node** named `Delete browser session in Firecrawl`.
    - Set **Resource** to `Browser`
    - Set **Operation** to `browserDelete`
    - Map `Session ID` from AI input
    - Connect it to the AI Agent as an `ai_tool`

18. **Configure credentials**:
    - **Firecrawl**
      - Sign up at [https://www.firecrawl.dev](https://www.firecrawl.dev)
      - Generate an API key
      - Attach that credential to every Firecrawl tool node
    - **OpenRouter**
      - Create an OpenRouter API key
      - Attach it to:
        - `Primary Chat Model`
        - `Fallback Chat Model`
        - `Parser Chat Model`

19. **Optionally add sticky notes** for maintainability:
    - `## Firecrawl Scrape Agent`
    - `## Input & Validation`
    - `## AI Agent & LLM Models`
    - `## Output & Response`
    - `## Firecrawl Tools`
    - `## Browser Tool`
    - `## Search & Scrape`

20. **Activate the workflow** so the production webhook URL becomes available.

21. **Test with a minimal POST request**:
    - Body:
      ```json
      {
        "prompt": "Find the pricing page for Firecrawl and extract the main plan names and prices."
      }
      ```

22. **Test with a schema-constrained request**:
    - Body:
      ```json
      {
        "prompt": "Find the pricing page for Firecrawl and extract the main plan names and prices.",
        "output_schema": {
          "type": "object",
          "properties": {
            "plans": {
              "type": "array"
            }
          }
        }
      }
      ```

23. **Expected behavior**:
    - If schema is missing: workflow uses permissive object output
    - If schema is invalid: immediate JSON error response
    - If prompt is valid: agent selects appropriate Firecrawl tools, then returns structured JSON

24. **Recommended hardening changes if rebuilding for production**:
    - explicitly validate `prompt` presence and type
    - add webhook authentication
    - add response status codes for validation failures
    - add limits or cleanup rules for browser session lifecycle
    - monitor token usage and Firecrawl quotas

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Firecrawl Scrape Agent: Send a POST request with a natural language prompt and an optional JSON schema. An AI agent uses Firecrawl to research the web and return clean, structured data matching your schema. | Workflow purpose |
| Use cases: data enrichment, lead generation, market research, competitor analysis, content aggregation. | Workflow purpose |
| Firecrawl setup requires an API key connected to all Firecrawl credential nodes. | [https://www.firecrawl.dev](https://www.firecrawl.dev) |
| LLM providers can be swapped if desired; the note mentions OpenRouter, OpenAI, and Anthropic as examples. | Model configuration guidance |
| Activate the workflow and copy the webhook URL before testing. | Deployment guidance |
| Test first with a simple POST body containing only `prompt`. | Validation guidance |

Disclaimer: The provided content comes exclusively from an automated workflow built with n8n, an integration and automation tool. This processing strictly respects applicable content policies and does not contain illegal, offensive, or protected material. All manipulated data is lawful and public.