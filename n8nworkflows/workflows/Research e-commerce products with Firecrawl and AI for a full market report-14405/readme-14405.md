Research e-commerce products with Firecrawl and AI for a full market report

https://n8nworkflows.xyz/workflows/research-e-commerce-products-with-firecrawl-and-ai-for-a-full-market-report-14405


# Research e-commerce products with Firecrawl and AI for a full market report

# 1. Workflow Overview

This workflow is an AI-powered product research assistant for e-commerce market analysis. A user submits a product research request via chat, the AI agent interprets the request, uses Firecrawl to search and scrape relevant comparison or marketplace pages, and then generates a structured market report covering prices, ratings, top products, customer pain points, and buying/selling recommendations.

It is designed for use cases such as:

- Researching the best products under a budget
- Comparing products across regional marketplaces
- Identifying market gaps for sellers
- Producing bilingual reports in English or Arabic
- Adapting search strategy based on country, marketplace, and currency hints

The workflow has a single entry point and is organized into the following logical blocks:

## 1.1 Input Reception

The workflow begins with a chat trigger that captures the user's request as conversational input.

## 1.2 AI Orchestration and Reasoning

A LangChain agent receives the user’s prompt, applies a detailed system instruction set, decides how to search, which sources to use, and how to synthesize the final report.

## 1.3 Memory and Model Support

The agent is supported by a buffer memory node for session continuity and a language model node via OpenRouter using Gemini 2.5 Flash.

## 1.4 External Research Tools

The agent can call two Firecrawl tools:
- a search tool to discover relevant pages or marketplace results
- a scrape tool to extract structured content from selected pages

## 1.5 Presentation and Embedded Notes

Sticky notes provide operational context, setup guidance, supported marketplaces, and usage examples. These notes do not affect execution but are important for maintainers and users.

---

# 2. Block-by-Block Analysis

## 2.1 Input Reception

### Overview

This block accepts the user’s message through the n8n chat interface and passes it into the research agent. It is the only execution entry point in this workflow.

### Nodes Involved

- `💬 Chat Input`

### Node Details

#### 💬 Chat Input

- **Type and technical role:** `@n8n/n8n-nodes-langchain.chatTrigger`  
  Entry trigger for conversational workflows. It waits for a user message in the n8n chat UI and starts execution.

- **Configuration choices:**  
  The node uses default options with no special customization. It simply captures the incoming chat message.

- **Key expressions or variables used:**  
  No custom expressions are configured in this node, but it outputs the user message in a structure consumed later as `chatInput`.

- **Input and output connections:**  
  - **Input:** none; it is the workflow trigger
  - **Output:** sends main output to `🧠 Product Research Agent`

- **Version-specific requirements:**  
  Uses node type version `1.4`. Requires an n8n version that supports the LangChain chat trigger node.

- **Edge cases or potential failure types:**  
  - Chat UI not opened or not supported in current environment
  - Missing/empty user message
  - Deployment environments where chat trigger is unavailable

- **Sub-workflow reference:**  
  None

---

## 2.2 AI Orchestration and Reasoning

### Overview

This is the core decision-making block. The agent interprets the user’s request, applies regional and marketplace logic, invokes Firecrawl tools as needed, and produces the final market analysis report.

### Nodes Involved

- `🧠 Product Research Agent`

### Node Details

#### 🧠 Product Research Agent

- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent`  
  Central AI agent node that coordinates prompt handling, memory usage, language model inference, and tool invocation.

- **Configuration choices:**  
  - Prompt source is explicitly defined
  - User input text is taken from the chat payload via expression
  - A long custom system message instructs the agent how to:
    - formulate search queries
    - prefer review/comparison pages over direct Amazon searches when appropriate
    - scrape only the top 2 results
    - build a concise report
    - adapt output language
    - handle Egypt, Saudi Arabia, UAE, or global contexts
    - infer currency expectations
    - avoid refusal and still provide best-effort analysis

- **Key expressions or variables used:**  
  - `={{ $json.chatInput }}`  
    This passes the incoming chat text into the agent as the main user prompt.

- **Input and output connections:**  
  - **Main input:** from `💬 Chat Input`
  - **AI language model input:** from `🤖 Gemini 2.5 Flash`
  - **AI memory input:** from `🧠 Conversation Memory`
  - **AI tool inputs:** from `🔍 Search Marketplaces` and `📄 Scrape Product Pages`
  - **Output:** final response is returned through the agent’s execution path to chat

- **Version-specific requirements:**  
  Uses node type version `3.1`. This requires an n8n version supporting LangChain agent tools, AI memory, and AI model attachment through dedicated connection types.

- **Edge cases or potential failure types:**  
  - If the chat payload does not contain `chatInput`, the prompt may be empty
  - Tool invocation may fail due to invalid credentials or Firecrawl API issues
  - LLM output quality may degrade if the user request is too vague
  - Long scraped content may increase token usage or cause truncation
  - Regional inference may be imperfect if the request mentions ambiguous locations or mixed marketplaces
  - The system message says “never say you cannot generate a report,” which may force low-confidence summaries when data is weak

- **Sub-workflow reference:**  
  None

---

## 2.3 Memory and Model Support

### Overview

This block provides conversational continuity and the underlying model used by the agent. The memory node maintains a session history keyed by a fixed custom identifier, and the OpenRouter model node supplies the actual LLM.

### Nodes Involved

- `🧠 Conversation Memory`
- `🤖 Gemini 2.5 Flash`

### Node Details

#### 🧠 Conversation Memory

- **Type and technical role:** `@n8n/n8n-nodes-langchain.memoryBufferWindow`  
  Memory component for the agent, storing prior interactions in a conversational window.

- **Configuration choices:**  
  - Uses a custom session ID type
  - Session key is hardcoded as `product-research-agent`

- **Key expressions or variables used:**  
  No dynamic expression is used here. The fixed session key is:
  - `product-research-agent`

- **Input and output connections:**  
  - **Input:** none through main execution
  - **AI memory output:** connected to `🧠 Product Research Agent`

- **Version-specific requirements:**  
  Uses node type version `1.3`. Requires LangChain memory support in your n8n version.

- **Edge cases or potential failure types:**  
  - Because the session key is static, different users may share memory context if deployed in a multi-user scenario
  - Old conversation context may pollute new requests
  - Memory retention behavior depends on node defaults and runtime environment

- **Sub-workflow reference:**  
  None

#### 🤖 Gemini 2.5 Flash

- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenRouter`  
  Chat model provider node using OpenRouter to access the Gemini 2.5 Flash model.

- **Configuration choices:**  
  - Model selected: `google/gemini-2.5-flash`
  - No extra model options are set

- **Key expressions or variables used:**  
  No expressions configured.

- **Input and output connections:**  
  - **Input:** none through main execution
  - **AI language model output:** connected to `🧠 Product Research Agent`

- **Version-specific requirements:**  
  Uses node type version `1`. Requires:
  - OpenRouter-compatible credential in n8n
  - Access to the specified model through the OpenRouter account

- **Edge cases or potential failure types:**  
  - Invalid or missing OpenRouter API key
  - Model unavailable in the account or region
  - Rate limits or usage caps
  - Output variability across providers or model revisions

- **Sub-workflow reference:**  
  None

---

## 2.4 External Research Tools

### Overview

This block gives the agent the ability to perform live research. The first tool searches for relevant review pages or marketplace results, and the second tool scrapes page content for downstream analysis.

### Nodes Involved

- `🔍 Search Marketplaces`
- `📄 Scrape Product Pages`

### Node Details

#### 🔍 Search Marketplaces

- **Type and technical role:** `@mendable/n8n-nodes-firecrawl.firecrawlTool`  
  Firecrawl tool node exposed to the AI agent for web search.

- **Configuration choices:**  
  - Resource: `MapSearch`
  - Operation: `search`
  - Query is supplied dynamically by the agent using an AI-tool parameter binding

- **Key expressions or variables used:**  
  - `={{ /*n8n-auto-generated-fromAI-override*/ $fromAI('Query', \`\`, 'string') }}`  
    This allows the AI agent to pass a search query string into the tool at runtime.

- **Input and output connections:**  
  - **Input:** none through main execution
  - **AI tool output:** connected to `🧠 Product Research Agent`

- **Version-specific requirements:**  
  Uses node type version `1`. Requires the Firecrawl community node package and valid Firecrawl credentials.

- **Edge cases or potential failure types:**  
  - Missing or invalid Firecrawl API key
  - Query generated by the agent may be too broad or too narrow
  - Search results may include irrelevant pages
  - Firecrawl service limits, latency, or outages
  - The `MapSearch` resource may behave differently depending on package version

- **Sub-workflow reference:**  
  None

#### 📄 Scrape Product Pages

- **Type and technical role:** `@mendable/n8n-nodes-firecrawl.firecrawlTool`  
  Firecrawl scraping tool exposed to the agent for extracting page content.

- **Configuration choices:**  
  - Operation: `scrape`
  - URL is supplied dynamically by the agent
  - Scrape output format is configured via `formats.format` with default/empty object selection, which typically means standard formatted extraction supported by the node
  - No custom headers are configured

- **Key expressions or variables used:**  
  - `={{ /*n8n-auto-generated-fromAI-override*/ $fromAI('URL', \`\`, 'string') }}`  
    Lets the AI agent provide the target URL at runtime.

- **Input and output connections:**  
  - **Input:** none through main execution
  - **AI tool output:** connected to `🧠 Product Research Agent`

- **Version-specific requirements:**  
  Uses node type version `1`. Requires Firecrawl credentials and a compatible installed Firecrawl node package.

- **Edge cases or potential failure types:**  
  - Invalid URL passed by the agent
  - Target page blocks scraping or is dynamically rendered in ways Firecrawl cannot fully parse
  - Content may be noisy, incomplete, or lacking consistent price/rating structure
  - Large pages may produce long outputs that stress token budgets
  - Marketplace pages may change layouts frequently

- **Sub-workflow reference:**  
  None

---

## 2.5 Presentation and Embedded Notes

### Overview

These nodes are non-executable annotations that document intent, setup, examples, and cost expectations. They are important for maintainers because they describe supported regions, expected behavior, and operational setup.

### Nodes Involved

- `Sticky Note`
- `Sticky Note1`
- `Sticky Note2`
- `Sticky Note3`

### Node Details

#### Sticky Note

- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Visual documentation node for workflow overview and setup instructions.

- **Configuration choices:**  
  Contains introductory text, setup guidance, author credit, and site reference.

- **Key expressions or variables used:**  
  None

- **Input and output connections:**  
  None

- **Version-specific requirements:**  
  Uses node type version `1`

- **Edge cases or potential failure types:**  
  None operational; purely visual

- **Sub-workflow reference:**  
  None

#### Sticky Note1

- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Visual note describing expected user inputs and example prompts.

- **Configuration choices:**  
  Styled with color variant `7`.

- **Key expressions or variables used:**  
  None

- **Input and output connections:**  
  None

- **Version-specific requirements:**  
  Uses node type version `1`

- **Edge cases or potential failure types:**  
  None operational

- **Sub-workflow reference:**  
  None

#### Sticky Note2

- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Visual note summarizing the AI agent’s responsibilities and supported logic.

- **Configuration choices:**  
  Styled with color variant `7`.

- **Key expressions or variables used:**  
  None

- **Input and output connections:**  
  None

- **Version-specific requirements:**  
  Uses node type version `1`

- **Edge cases or potential failure types:**  
  None operational

- **Sub-workflow reference:**  
  None

#### Sticky Note3

- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Visual note documenting the Firecrawl tools and rough credit consumption.

- **Configuration choices:**  
  Styled with color variant `7`.

- **Key expressions or variables used:**  
  None

- **Input and output connections:**  
  None

- **Version-specific requirements:**  
  Uses node type version `1`

- **Edge cases or potential failure types:**  
  None operational

- **Sub-workflow reference:**  
  None

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| 💬 Chat Input | `@n8n/n8n-nodes-langchain.chatTrigger` | Chat-based workflow entry point |  | 🧠 Product Research Agent | 💬 INPUT<br>━━━━━━<br>User sends a product research request via chat.<br><br>Examples:<br>- "Research wireless earbuds under $30"<br>- "ابحثلي عن أحسن ماكينة قهوة في مصر أقل من 2500 جنيه"<br>- "Find best phone cases on Amazon" |
| 🧠 Product Research Agent | `@n8n/n8n-nodes-langchain.agent` | Main AI orchestrator that plans, calls tools, and generates the report | 💬 Chat Input; 🤖 Gemini 2.5 Flash; 🧠 Conversation Memory; 🔍 Search Marketplaces; 📄 Scrape Product Pages |  | 🧠 AI AGENT<br>━━━━━━━━━<br>Orchestrates the full research workflow:<br><br>1. Receives user query<br>2. Plans search strategy based on region/marketplace<br>3. Calls Firecrawl tools to gather data<br>4. Analyzes results with AI<br>5. Generates structured report<br><br>Supports:<br>- Automatic marketplace detection by country<br>- Currency handling (USD, EGP, SAR, AED)<br>- Bilingual responses (EN/AR) |
| 🧠 Conversation Memory | `@n8n/n8n-nodes-langchain.memoryBufferWindow` | Stores conversation context for the agent |  | 🧠 Product Research Agent |  |
| 🔍 Search Marketplaces | `@mendable/n8n-nodes-firecrawl.firecrawlTool` | AI tool for Firecrawl search queries |  | 🧠 Product Research Agent | 🔥 FIRECRAWL TOOLS<br>━━━━━━━━━━━━━━━━<br>Search: Finds product pages across the web and marketplaces<br><br>Scrape: Extracts detailed product data (name, price, rating, reviews)<br>in clean markdown format<br><br>💡 Credits: ~8-10 per research<br>(1 search + 2 scrapes) |
| 📄 Scrape Product Pages | `@mendable/n8n-nodes-firecrawl.firecrawlTool` | AI tool for scraping selected result pages |  | 🧠 Product Research Agent | 🔥 FIRECRAWL TOOLS<br>━━━━━━━━━━━━━━━━<br>Search: Finds product pages across the web and marketplaces<br><br>Scrape: Extracts detailed product data (name, price, rating, reviews)<br>in clean markdown format<br><br>💡 Credits: ~8-10 per research<br>(1 search + 2 scrapes) |
| Sticky Note | `n8n-nodes-base.stickyNote` | Visual workflow overview and setup note |  |  | 📦 AI Product Research Agent<br>━━━━━━━━━━━━━━━━━━━━━━━━━<br>Built for the n8n x Firecrawl Community Challenge (April 2026)<br><br>🔍 What it does:<br>Researches any product across global & regional marketplaces and generates a comprehensive market analysis report.<br><br>✨ Features:<br>- Multi-marketplace: Amazon, Noon, Jumia, AliExpress & more<br>- Regional awareness: Egypt, Saudi Arabia, UAE, Global<br>- Bilingual: English & Arabic (Egyptian dialect)<br>- Full report: Pricing, ratings, insights, complaints & recommendations<br>- Market gap analysis for sellers<br><br>🛠 Setup (3 steps):<br>1. Add your Firecrawl API key (free at firecrawl.dev)<br>2. Add your OpenAI/OpenRouter API key<br>3. Click "Open Chat" and start researching!<br><br>👤 Built by: Osama Goda (@osamagoda)<br>🌐 makeaiagents.dev |
| Sticky Note1 | `n8n-nodes-base.stickyNote` | Visual note for input examples |  |  | 💬 INPUT<br>━━━━━━<br>User sends a product research request via chat.<br><br>Examples:<br>- "Research wireless earbuds under $30"<br>- "ابحثلي عن أحسن ماكينة قهوة في مصر أقل من 2500 جنيه"<br>- "Find best phone cases on Amazon" |
| Sticky Note2 | `n8n-nodes-base.stickyNote` | Visual note for AI agent behavior |  |  | 🧠 AI AGENT<br>━━━━━━━━━<br>Orchestrates the full research workflow:<br><br>1. Receives user query<br>2. Plans search strategy based on region/marketplace<br>3. Calls Firecrawl tools to gather data<br>4. Analyzes results with AI<br>5. Generates structured report<br><br>Supports:<br>- Automatic marketplace detection by country<br>- Currency handling (USD, EGP, SAR, AED)<br>- Bilingual responses (EN/AR) |
| Sticky Note3 | `n8n-nodes-base.stickyNote` | Visual note for Firecrawl usage and credits |  |  | 🔥 FIRECRAWL TOOLS<br>━━━━━━━━━━━━━━━━<br>Search: Finds product pages across the web and marketplaces<br><br>Scrape: Extracts detailed product data (name, price, rating, reviews)<br>in clean markdown format<br><br>💡 Credits: ~8-10 per research<br>(1 search + 2 scrapes) |
| 🤖 Gemini 2.5 Flash | `@n8n/n8n-nodes-langchain.lmChatOpenRouter` | LLM backing the agent through OpenRouter |  | 🧠 Product Research Agent |  |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it something like: `AI Product Research Agent powered by Firecrawl`
   - Leave it inactive until credentials are configured and tested.

2. **Add the chat trigger node**
   - Create node: `Chat Trigger`
   - Node type: `@n8n/n8n-nodes-langchain.chatTrigger`
   - Name it: `💬 Chat Input`
   - Keep default options unless your deployment requires specific chat settings.

3. **Add the main AI agent node**
   - Create node: `AI Agent`
   - Node type: `@n8n/n8n-nodes-langchain.agent`
   - Name it: `🧠 Product Research Agent`
   - Set the prompt type to a manually defined prompt
   - In the main text/input field, use:
     - `={{ $json.chatInput }}`
   - In the system message field, paste the full agent instruction set below in substance:
     - product research specialization
     - search using review/comparison intent
     - avoid direct Amazon search unless marketplace-specific request implies it
     - scrape only top 2 results
     - produce sections for market overview, top 3 products, key insights, common complaints, recommendation
     - respond in same language as user
     - use Egyptian Arabic if Arabic input
     - always include source URLs
     - adapt by region and currency
     - marketplace guide for Egypt, Saudi Arabia, UAE, and global
   - Preserve the logic from the original workflow if you want equivalent behavior.

4. **Connect the chat trigger to the agent**
   - Main connection:
     - `💬 Chat Input` → `🧠 Product Research Agent`

5. **Add the conversation memory node**
   - Create node: `Memory Buffer Window`
   - Node type: `@n8n/n8n-nodes-langchain.memoryBufferWindow`
   - Name it: `🧠 Conversation Memory`
   - Set:
     - `Session ID Type` = custom key
     - `Session Key` = `product-research-agent`
   - Connect it to the agent using an **AI Memory** connection:
     - `🧠 Conversation Memory` → `🧠 Product Research Agent`

6. **Review the memory design before production**
   - The original workflow uses a single static session key.
   - If you expect multiple users, replace the fixed value with a user-specific expression to avoid memory collisions.
   - Example strategy:
     - use chat session ID if available
     - or use a user ID from the trigger payload

7. **Add the language model node**
   - Create node: `OpenRouter Chat Model`
   - Node type: `@n8n/n8n-nodes-langchain.lmChatOpenRouter`
   - Name it: `🤖 Gemini 2.5 Flash`
   - Select model:
     - `google/gemini-2.5-flash`
   - Leave advanced options empty unless you want to control temperature, max tokens, or other inference settings
   - Add OpenRouter credentials:
     - create/select an `OpenRouter API` credential
     - enter your OpenRouter API key
   - Connect it to the agent using an **AI Language Model** connection:
     - `🤖 Gemini 2.5 Flash` → `🧠 Product Research Agent`

8. **Add the Firecrawl search tool**
   - Create node using the Firecrawl community integration
   - Node type: `@mendable/n8n-nodes-firecrawl.firecrawlTool`
   - Name it: `🔍 Search Marketplaces`
   - Set:
     - `Resource` = `MapSearch`
     - `Operation` = `search`
   - In the query field, use the AI parameter expression:
     - `={{ $fromAI('Query', '', 'string') }}`
   - Add Firecrawl credentials:
     - create/select `firecrawlApi`
     - enter your Firecrawl API key
   - Connect it to the agent using an **AI Tool** connection:
     - `🔍 Search Marketplaces` → `🧠 Product Research Agent`

9. **Add the Firecrawl scrape tool**
   - Create another Firecrawl tool node
   - Node type: `@mendable/n8n-nodes-firecrawl.firecrawlTool`
   - Name it: `📄 Scrape Product Pages`
   - Set:
     - `Operation` = `scrape`
   - In the URL field, use:
     - `={{ $fromAI('URL', '', 'string') }}`
   - In scrape options:
     - keep standard formatting enabled as in the original workflow
     - leave headers empty unless target sites require specific headers
   - Use the same Firecrawl credential as the search node
   - Connect it to the agent using an **AI Tool** connection:
     - `📄 Scrape Product Pages` → `🧠 Product Research Agent`

10. **Ensure all AI-specific connections are correct**
    - You should have:
      - Main: `💬 Chat Input` → `🧠 Product Research Agent`
      - AI Language Model: `🤖 Gemini 2.5 Flash` → `🧠 Product Research Agent`
      - AI Memory: `🧠 Conversation Memory` → `🧠 Product Research Agent`
      - AI Tool: `🔍 Search Marketplaces` → `🧠 Product Research Agent`
      - AI Tool: `📄 Scrape Product Pages` → `🧠 Product Research Agent`

11. **Add the visual notes if you want parity with the original**
    - Add four `Sticky Note` nodes.
    - Suggested content:

    **Sticky note A**
    - Overview of the product research agent
    - Mention Firecrawl and OpenRouter setup
    - Mention supported marketplaces and bilingual behavior
    - Credit the author if desired

    **Sticky note B**
    - Example user prompts:
      - `Research wireless earbuds under $30`
      - `ابحثلي عن أحسن ماكينة قهوة في مصر أقل من 2500 جنيه`
      - `Find best phone cases on Amazon`

    **Sticky note C**
    - Explain that the AI agent:
      - receives query
      - plans search strategy
      - calls Firecrawl
      - analyzes results
      - generates report

    **Sticky note D**
    - Explain Firecrawl usage:
      - one search plus two scrapes
      - approximate credit cost

12. **Set workflow execution order**
    - In workflow settings, set execution order to `v1` to match the original configuration.

13. **Configure credentials**
    - **Firecrawl**
      - Create Firecrawl API credential
      - Paste your Firecrawl API key
    - **OpenRouter**
      - Create OpenRouter credential
      - Paste your API key
      - Verify that `google/gemini-2.5-flash` is accessible on your account

14. **Test the workflow with sample prompts**
    - Example tests:
      - `Research wireless earbuds under $30`
      - `Find the best office chairs in Saudi Arabia under 500 SAR`
      - `ابحثلي عن أحسن سماعات بلوتوث في مصر أقل من 1500 جنيه`
      - `Find best phone cases on Amazon`
    - Confirm the agent:
      - issues a sensible search query
      - calls the search tool
      - scrapes at most two pages
      - returns a report with source URLs

15. **Validate expected output structure**
    - The final answer should usually contain:
      - Market overview
      - Top 3 products
      - Key insights
      - Common complaints
      - Recommendation
      - Source URLs

16. **Harden the workflow if deploying to production**
    - Replace static memory key with per-user session key
    - Consider setting model limits or temperature
    - Add fallback handling if Firecrawl search returns poor results
    - Add moderation or validation if users can submit arbitrary URLs or broad research requests
    - Consider using a structured output parser if downstream automation depends on consistent report formatting

17. **Optional enhancements**
    - Add a code or set node after the agent to convert the report into JSON sections
    - Add Google Sheets, Notion, or database storage
    - Add PDF generation for downloadable reports
    - Add region-specific post-processing for currency normalization
    - Add a second model for validation or summarization

### Sub-workflow setup

There are **no sub-workflows** in this workflow. No `Execute Workflow` node or workflow-calling tool is present.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Built for the n8n x Firecrawl Community Challenge (April 2026) | Project context |
| Firecrawl API key is required for search and scrape operations | `https://firecrawl.dev` |
| OpenRouter API key is required for the Gemini model node | `https://openrouter.ai` |
| Author credit: Osama Goda (@osamagoda) | Workflow note |
| Website reference: makeaiagents.dev | `https://makeaiagents.dev` |
| Estimated Firecrawl usage is about 8–10 credits per research run | Operational estimate from sticky note |

