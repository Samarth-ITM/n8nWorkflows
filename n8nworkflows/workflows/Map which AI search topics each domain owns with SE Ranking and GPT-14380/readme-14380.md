Map which AI search topics each domain owns with SE Ranking and GPT

https://n8nworkflows.xyz/workflows/map-which-ai-search-topics-each-domain-owns-with-se-ranking-and-gpt-14380


# Map which AI search topics each domain owns with SE Ranking and GPT

## 1. Workflow Overview

This workflow collects AI-search prompt data for one primary domain and up to two competitors using SE Ranking, groups those prompts into thematic clusters with OpenAI, identifies which domain “owns” each topic, and appends the resulting topic analysis to Google Sheets.

Typical use cases:
- SEO competitive intelligence
- AI visibility benchmarking across brands/domains
- Editorial planning based on topic gaps
- Tracking topic-level dominance in AI search ecosystems

### 1.1 Input Reception and Normalization
The workflow starts with a hosted form where the user submits:
- their domain and brand
- up to two competitor domains and brands
- a target market database

The submitted values are normalized into internal variables in a Set node.

### 1.2 Prompt Collection from SE Ranking
The workflow fans out into six parallel branches:
- your domain target prompts
- your brand prompts
- competitor 1 target prompts
- competitor 1 brand prompts
- competitor 2 target prompts
- competitor 2 brand prompts

Each branch adds a different wait delay before calling the SE Ranking community node. This is clearly designed to reduce rate-limit pressure.

### 1.3 Prompt Consolidation and Filtering
All six SE Ranking responses are merged. A Code node then:
- maps each response to the correct domain/brand context
- extracts prompt text
- filters competitor prompts to SEO-related terms only
- prepares a single combined text payload for GPT

### 1.4 GPT Topic Clustering and Reasoning
An OpenAI node receives the formatted prompt inventory and a system instruction that asks the model to:
- cluster prompts into 5–10 SEO topics
- count prompts per domain for each topic
- identify a winner per topic
- produce an overall winner and summary
- return strict JSON only

### 1.5 Output Parsing and Export
A final Code node parses GPT output into row-oriented records, then appends those rows into a Google Sheet tab for downstream reporting.

---

## 2. Block-by-Block Analysis

## 2.1 Input Reception and Configuration

**Overview:**  
This block exposes a public form entry point and converts user-submitted values into stable internal fields used by all downstream nodes. It also provides fallback logic so competitor 2 defaults to competitor 1 if not supplied.

**Nodes Involved:**
- Domain Input Form
- Configuration

### Node: Domain Input Form
- **Type and technical role:** `n8n-nodes-base.formTrigger`  
  Entry-point node that hosts a form and triggers the workflow on submission.
- **Configuration choices:**
  - Form title: `AI Search Topic Analysis`
  - Description: asks for domain plus up to 2 competitors to analyze AI-search topic ownership
  - Required fields:
    - Your Domain
    - Your Brand Name
    - Competitor 1 Domain
    - Competitor 1 Brand
    - Target Market
  - Optional fields:
    - Competitor 2 Domain
    - Competitor 2 Brand
  - Target Market dropdown options:
    - us, uk, de, fr, es, it, au, ca, pl
- **Key expressions or variables used:** none
- **Input and output connections:**
  - Entry node
  - Outputs to `Configuration`
- **Version-specific requirements (if any):**
  - Type version `2.2`
  - Requires a recent n8n version with Form Trigger support
- **Edge cases or potential failure types:**
  - Workflow must be active to use the production form URL
  - Optional competitor 2 fields may be blank
  - User may enter malformed domains or brand names; no validation/sanitization is implemented
- **Sub-workflow reference:** none

### Node: Configuration
- **Type and technical role:** `n8n-nodes-base.set`  
  Normalizes form fields into reusable internal keys.
- **Configuration choices:**
  - Sets:
    - `your_domain` from `Your Domain`
    - `your_brand` from `Your Brand Name`
    - `competitor_domain_1` from `Competitor 1 Domain`
    - `competitor_brand_1` from `Competitor 1 Brand`
    - `competitor_domain_2` defaults to competitor 1 domain if competitor 2 is blank
    - `competitor_brand_2` defaults to competitor 1 brand if competitor 2 is blank
    - `source` from `Target Market`
    - `scope` fixed to `base_domain`
    - `prompts_limit` fixed to `10`
- **Key expressions or variables used:**
  - `{{ $json['Your Domain'] }}`
  - `{{ $json['Competitor 2 Domain'] || $json['Competitor 1 Domain'] }}`
  - `{{ $json['Target Market'] }}`
- **Input and output connections:**
  - Input from `Domain Input Form`
  - Outputs to all six wait nodes:
    - `Wait (3s)`
    - `Wait (6s)`
    - `Wait (9s)`
    - `Wait (12s)`
    - `Wait (15s)`
    - `Wait (18s)`
- **Version-specific requirements (if any):**
  - Type version `3`
- **Edge cases or potential failure types:**
  - If competitor 2 is omitted, competitor 1 is duplicated as competitor 2, which can skew competitive counts
  - `prompts_limit` is stored as a string, not a number; many nodes will coerce it correctly, but this is worth monitoring
  - Domain values are not normalized to lowercase or stripped of protocol prefixes
- **Sub-workflow reference:** none

---

## 2.2 Prompt Collection for the Primary Domain

**Overview:**  
This block fetches AI-search prompts for the primary domain both by target domain and by brand name. The staggered delays reduce the chance of API throttling.

**Nodes Involved:**
- Wait (3s)
- Get your target prompts
- Wait (6s)
- Get your brand prompts

### Node: Wait (3s)
- **Type and technical role:** `n8n-nodes-base.wait`  
  Delays execution before the first SE Ranking request.
- **Configuration choices:**
  - Wait amount: `3`
- **Key expressions or variables used:** none
- **Input and output connections:**
  - Input from `Configuration`
  - Output to `Get your target prompts`
- **Version-specific requirements (if any):**
  - Type version `1.1`
- **Edge cases or potential failure types:**
  - Minimal delay may still be insufficient if API rate limits are strict
- **Sub-workflow reference:** none

### Node: Get your target prompts
- **Type and technical role:** `@seranking/n8n-nodes-seranking.seRanking`  
  Calls SE Ranking AI Search API to fetch prompts where the submitted domain appears as a target.
- **Configuration choices:**
  - Resource: `aiSearch`
  - Operation: `getPromptsByTarget`
  - Engine: `ai-overview`
  - Scope: from configuration (`base_domain`)
  - Domain: `your_domain`
  - Source: regional database from form
  - Additional fields:
    - sort: `volume`
    - limit: `prompts_limit`
    - sort order: `desc`
- **Key expressions or variables used:**
  - `{{ $('Configuration').item.json.scope }}`
  - `{{ $('Configuration').item.json.your_domain }}`
  - `{{ $('Configuration').item.json.source }}`
  - `{{ $('Configuration').item.json.prompts_limit }}`
- **Input and output connections:**
  - Input from `Wait (3s)`
  - Output to `Merge all prompts` input 0
- **Version-specific requirements (if any):**
  - Type version `1`
  - Requires SE Ranking community node package installed
- **Edge cases or potential failure types:**
  - Missing or invalid SE Ranking API credentials
  - Unsupported region/source value
  - No prompt data returned
  - API throttling or transient failure
  - Domain formatting issues
- **Sub-workflow reference:** none

### Node: Wait (6s)
- **Type and technical role:** `n8n-nodes-base.wait`
- **Configuration choices:**
  - Wait amount: `6`
- **Key expressions or variables used:** none
- **Input and output connections:**
  - Input from `Configuration`
  - Output to `Get your brand prompts`
- **Version-specific requirements (if any):**
  - Type version `1.1`
- **Edge cases or potential failure types:**
  - Delay helps pacing but does not guarantee no rate limit
- **Sub-workflow reference:** none

### Node: Get your brand prompts
- **Type and technical role:** `@seranking/n8n-nodes-seranking.seRanking`  
  Fetches prompts where the submitted brand name appears in AI search.
- **Configuration choices:**
  - Resource: `aiSearch`
  - Operation: `getPromptsByBrand`
  - Engine: `ai-overview`
  - Brand name: `your_brand`
  - Source: regional database
  - Additional fields:
    - sort: `volume`
    - limit: `prompts_limit`
    - sort order: `desc`
- **Key expressions or variables used:**
  - `{{ $('Configuration').item.json.your_brand }}`
  - `{{ $('Configuration').item.json.source }}`
  - `{{ $('Configuration').item.json.prompts_limit }}`
- **Input and output connections:**
  - Input from `Wait (6s)`
  - Output to `Merge all prompts` input 1
- **Version-specific requirements (if any):**
  - Type version `1`
  - Requires SE Ranking community node package
- **Edge cases or potential failure types:**
  - Brand ambiguities may pull irrelevant prompts
  - Empty brand values would likely fail or return no results, though form requires this field
- **Sub-workflow reference:** none

---

## 2.3 Prompt Collection for Competitor 1

**Overview:**  
This block fetches target-domain and brand-level prompts for competitor 1. Those prompts are later filtered for SEO relevance before being sent to GPT.

**Nodes Involved:**
- Wait (9s)
- Get competitor 1 target prompts
- Wait (12s)
- Get competitor 1 brand prompts

### Node: Wait (9s)
- **Type and technical role:** `n8n-nodes-base.wait`
- **Configuration choices:**
  - Wait amount: `9`
- **Key expressions or variables used:** none
- **Input and output connections:**
  - Input from `Configuration`
  - Output to `Get competitor 1 target prompts`
- **Version-specific requirements (if any):**
  - Type version `1.1`
- **Edge cases or potential failure types:** standard wait-node considerations
- **Sub-workflow reference:** none

### Node: Get competitor 1 target prompts
- **Type and technical role:** `@seranking/n8n-nodes-seranking.seRanking`
- **Configuration choices:**
  - Resource: `aiSearch`
  - Operation: `getPromptsByTarget`
  - Engine: `ai-overview`
  - Scope: `base_domain`
  - Domain: `competitor_domain_1`
  - Source: selected market
  - Additional fields:
    - sort: `volume`
    - limit: `prompts_limit`
    - sort order: `desc`
- **Key expressions or variables used:**
  - `{{ $('Configuration').item.json.competitor_domain_1 }}`
  - `{{ $('Configuration').item.json.scope }}`
  - `{{ $('Configuration').item.json.source }}`
- **Input and output connections:**
  - Input from `Wait (9s)`
  - Output to `Merge all prompts` input 2
- **Version-specific requirements (if any):**
  - Type version `1`
- **Edge cases or potential failure types:**
  - Invalid competitor domain
  - Empty input impossible if form is completed as intended because competitor 1 is required
  - API rate or auth issues
- **Sub-workflow reference:** none

### Node: Wait (12s)
- **Type and technical role:** `n8n-nodes-base.wait`
- **Configuration choices:**
  - Wait amount: `12`
- **Key expressions or variables used:** none
- **Input and output connections:**
  - Input from `Configuration`
  - Output to `Get competitor 1 brand prompts`
- **Version-specific requirements (if any):**
  - Type version `1.1`
- **Edge cases or potential failure types:** standard wait-node considerations
- **Sub-workflow reference:** none

### Node: Get competitor 1 brand prompts
- **Type and technical role:** `@seranking/n8n-nodes-seranking.seRanking`
- **Configuration choices:**
  - Resource: `aiSearch`
  - Operation: `getPromptsByBrand`
  - Engine: `ai-overview`
  - Brand name: `competitor_brand_1`
  - Source: selected market
  - Additional fields:
    - sort: `volume`
    - limit: `prompts_limit`
    - sort order: `desc`
- **Key expressions or variables used:**
  - `{{ $('Configuration').item.json.competitor_brand_1 }}`
  - `{{ $('Configuration').item.json.source }}`
- **Input and output connections:**
  - Input from `Wait (12s)`
  - Output to `Merge all prompts` input 3
- **Version-specific requirements (if any):**
  - Type version `1`
- **Edge cases or potential failure types:**
  - Brand ambiguity and false positives
  - API/auth/rate-limit issues
- **Sub-workflow reference:** none

---

## 2.4 Prompt Collection for Competitor 2

**Overview:**  
This block mirrors competitor 1 collection for a second competitor. If competitor 2 is omitted in the form, this branch uses competitor 1 as fallback, effectively duplicating it.

**Nodes Involved:**
- Wait (15s)
- Get competitor 2 target prompts
- Wait (18s)
- Get competitor 2 brand prompts

### Node: Wait (15s)
- **Type and technical role:** `n8n-nodes-base.wait`
- **Configuration choices:**
  - Wait amount: `15`
- **Key expressions or variables used:** none
- **Input and output connections:**
  - Input from `Configuration`
  - Output to `Get competitor 2 target prompts`
- **Version-specific requirements (if any):**
  - Type version `1.1`
- **Edge cases or potential failure types:** standard wait-node considerations
- **Sub-workflow reference:** none

### Node: Get competitor 2 target prompts
- **Type and technical role:** `@seranking/n8n-nodes-seranking.seRanking`
- **Configuration choices:**
  - Resource: `aiSearch`
  - Operation: `getPromptsByTarget`
  - Engine: `ai-overview`
  - Scope: `base_domain`
  - Domain: `competitor_domain_2`
  - Source: selected market
  - Additional fields:
    - sort: `volume`
    - limit: `prompts_limit`
    - sort order: `desc`
- **Key expressions or variables used:**
  - `{{ $('Configuration').item.json.competitor_domain_2 }}`
  - `{{ $('Configuration').item.json.scope }}`
  - `{{ $('Configuration').item.json.source }}`
- **Input and output connections:**
  - Input from `Wait (15s)`
  - Output to `Merge all prompts` input 4
- **Version-specific requirements (if any):**
  - Type version `1`
- **Edge cases or potential failure types:**
  - If competitor 2 is blank, this uses competitor 1 fallback
  - Can produce duplicated analysis disguised as two competitors
- **Sub-workflow reference:** none

### Node: Wait (18s)
- **Type and technical role:** `n8n-nodes-base.wait`
- **Configuration choices:**
  - Wait amount: `18`
- **Key expressions or variables used:** none
- **Input and output connections:**
  - Input from `Configuration`
  - Output to `Get competitor 2 brand prompts`
- **Version-specific requirements (if any):**
  - Type version `1.1`
- **Edge cases or potential failure types:** standard wait-node considerations
- **Sub-workflow reference:** none

### Node: Get competitor 2 brand prompts
- **Type and technical role:** `@seranking/n8n-nodes-seranking.seRanking`
- **Configuration choices:**
  - Resource: `aiSearch`
  - Operation: `getPromptsByBrand`
  - Engine: `ai-overview`
  - Brand name: `competitor_brand_2`
  - Source: selected market
  - Additional fields:
    - sort: `volume`
    - limit: `prompts_limit`
    - sort order: `desc`
- **Key expressions or variables used:**
  - `{{ $('Configuration').item.json.competitor_brand_2 }}`
  - `{{ $('Configuration').item.json.source }}`
- **Input and output connections:**
  - Input from `Wait (18s)`
  - Output to `Merge all prompts` input 5
- **Version-specific requirements (if any):**
  - Type version `1`
- **Edge cases or potential failure types:**
  - Duplicated competitor 1 fallback if competitor 2 omitted
  - Brand ambiguity and API issues
- **Sub-workflow reference:** none

---

## 2.5 Prompt Consolidation and GPT Input Preparation

**Overview:**  
This block merges the six prompt datasets and converts them into a GPT-friendly text representation. It also applies SEO-related filtering to competitor prompts to reduce noise.

**Nodes Involved:**
- Merge all prompts
- Format prompts for GPT

### Node: Merge all prompts
- **Type and technical role:** `n8n-nodes-base.merge`  
  Collects six separate upstream results into one combined execution context.
- **Configuration choices:**
  - Number of inputs: `6`
- **Key expressions or variables used:** none
- **Input and output connections:**
  - Inputs:
    - input 0: `Get your target prompts`
    - input 1: `Get your brand prompts`
    - input 2: `Get competitor 1 target prompts`
    - input 3: `Get competitor 1 brand prompts`
    - input 4: `Get competitor 2 target prompts`
    - input 5: `Get competitor 2 brand prompts`
  - Output to `Format prompts for GPT`
- **Version-specific requirements (if any):**
  - Type version `3.2`
- **Edge cases or potential failure types:**
  - If any upstream branch errors, the merge will not complete
  - Behavior depends on shape of incoming items; here it is expected each input returns one item containing a `prompts` array
- **Sub-workflow reference:** none

### Node: Format prompts for GPT
- **Type and technical role:** `n8n-nodes-base.code`  
  Transforms merged API outputs into one text block for LLM clustering.
- **Configuration choices (interpreted):**
  - Reads all six merged inputs with `$input.all()`
  - Reads configuration from the `Configuration` node
  - Defines a list of SEO terms used to identify SEO-related prompts
  - Builds a `domainMap` matching each input index to:
    - your domain target
    - your domain brand
    - competitor 1 target
    - competitor 1 brand
    - competitor 2 target
    - competitor 2 brand
  - Extracts `prompt` text from `input.json.prompts`
  - Applies SEO filtering only to competitor data (`filterSeo: true`)
  - Joins remaining prompts into sections like:
    - `Domain: example.com (your domain — target)`
    - `Prompts: ...`
  - Returns:
    - `prompt_data`
    - all domain/brand config values
- **Key expressions or variables used:**
  - `$input.all()`
  - `$('Configuration').first().json`
  - `input.json.prompts`
- **Input and output connections:**
  - Input from `Merge all prompts`
  - Output to `GPT topic clustering`
- **Version-specific requirements (if any):**
  - Type version `2`
- **Edge cases or potential failure types:**
  - Assumes each SE Ranking response has a `prompts` array with objects containing `prompt`
  - Competitor filtering may remove all prompts for a competitor
  - Own-domain prompts are not filtered, so topic balance may be asymmetrical
  - If order of merge inputs changes, labeling becomes incorrect because mapping is index-based
  - SEO keyword filter is simplistic and may exclude valid prompts or include noisy ones
- **Sub-workflow reference:** none

---

## 2.6 GPT Topic Clustering

**Overview:**  
This block sends the consolidated prompt text to OpenAI and asks the model to generate a structured topic ownership analysis in JSON.

**Nodes Involved:**
- GPT topic clustering

### Node: GPT topic clustering
- **Type and technical role:** `@n8n/n8n-nodes-langchain.openAi`  
  Calls OpenAI through the LangChain-compatible n8n node.
- **Configuration choices:**
  - Model: `gpt-5.4-nano-2026-03-17`
  - Temperature: `0.3`
  - Input messages:
    - User/content message: `{{ $json.prompt_data }}`
    - System message: instructs the model to:
      - cluster prompts into 5–10 topics
      - count prompts per domain
      - identify winner per topic
      - write actionable insight
      - include representative prompts from all domains
      - return valid JSON only
  - Text output format options enabled
- **Key expressions or variables used:**
  - `{{ $json.prompt_data }}`
- **Input and output connections:**
  - Input from `Format prompts for GPT`
  - Output to `Format GPT output`
- **Version-specific requirements (if any):**
  - Type version `2.1`
  - Requires OpenAI credentials configured in n8n
  - Uses a specific model ID that may not exist in all accounts or future environments
- **Edge cases or potential failure types:**
  - Invalid or unavailable model ID
  - OpenAI auth or quota failure
  - Non-JSON output despite instructions
  - Hallucinated domain assignments
  - Topic count fields may not align with actual prompt distributions
- **Sub-workflow reference:** none

---

## 2.7 GPT Output Parsing and Spreadsheet Export

**Overview:**  
This block parses the LLM response, converts topic objects into flat row records, and appends them to Google Sheets.

**Nodes Involved:**
- Format GPT output
- Export to Sheets: Topic Analysis

### Node: Format GPT output
- **Type and technical role:** `n8n-nodes-base.code`  
  Parses OpenAI output JSON and maps it into row-wise objects.
- **Configuration choices (interpreted):**
  - Reads the first input item
  - Attempts to locate text content from multiple possible OpenAI response shapes:
    - `output[0].content[0].text`
    - `message.content`
    - `choices[0].message.content`
  - Strips code fences like ```json
  - Parses JSON
  - On parse failure, returns one item with:
    - `error`
    - `raw`
  - For each topic, emits:
    - `topic`
    - `winner`
    - `<your_domain>_prompts`
    - `<competitor_domain_1>_prompts`
    - `<competitor_domain_2>_prompts`
    - `insight`
    - `top_prompts`
    - `overall_winner`
    - `summary`
    - `date`
- **Key expressions or variables used:**
  - `$input.first().json`
  - `$('Configuration').first().json`
  - dynamic keys based on domain values
- **Input and output connections:**
  - Input from `GPT topic clustering`
  - Output to `Export to Sheets: Topic Analysis`
- **Version-specific requirements (if any):**
  - Type version `2`
- **Edge cases or potential failure types:**
  - JSON parse failures
  - GPT may omit expected keys such as `topics`
  - Dynamic field names based on domain strings may produce awkward spreadsheet headers if domains contain special characters
  - If parse fails, the error object still flows to Google Sheets unless manually filtered
- **Sub-workflow reference:** none

### Node: Export to Sheets: Topic Analysis
- **Type and technical role:** `n8n-nodes-base.googleSheets`  
  Appends the formatted analysis rows to a Google Sheet.
- **Configuration choices:**
  - Operation: `append`
  - Sheet name selected as `gid=0`
  - Document ID expected from spreadsheet URL, but currently blank in the workflow JSON
- **Key expressions or variables used:** none
- **Input and output connections:**
  - Input from `Format GPT output`
  - No downstream node
- **Version-specific requirements (if any):**
  - Type version `4.4`
  - Requires Google Sheets OAuth2 credentials
- **Edge cases or potential failure types:**
  - Spreadsheet URL/document ID is not configured, so this node will fail until set
  - OAuth permission issues
  - Header mismatch when dynamic column names change between runs
  - Append behavior may create inconsistent schemas if different domains are analyzed across executions
- **Sub-workflow reference:** none

---

## 2.8 Documentation and In-Canvas Notes

**Overview:**  
These sticky notes document the workflow’s business purpose, setup requirements, sharing instructions, and functional grouping. They do not affect execution but are important for maintainability.

**Nodes Involved:**
- Sticky Note1
- Sticky Note
- Sticky Note4
- Sticky Note2
- Sticky Note3

### Node: Sticky Note1
- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Documentation panel describing the workflow’s purpose, setup, requirements, and customization.
- **Configuration choices:** large overview note with links to SE Ranking API dashboard and npm package
- **Key expressions or variables used:** none
- **Input and output connections:** none
- **Version-specific requirements (if any):**
  - Type version `1`
- **Edge cases or potential failure types:** none
- **Sub-workflow reference:** none

### Node: Sticky Note
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:** labels competitor prompt collection area
- **Input and output connections:** none
- **Version-specific requirements (if any):** type version `1`
- **Edge cases or potential failure types:** none
- **Sub-workflow reference:** none

### Node: Sticky Note4
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:** explains how to share/embed the form and includes iframe example
- **Input and output connections:** none
- **Version-specific requirements (if any):** type version `1`
- **Edge cases or potential failure types:** none
- **Sub-workflow reference:** none

### Node: Sticky Note2
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:** labels primary-domain prompt collection area
- **Input and output connections:** none
- **Version-specific requirements (if any):** type version `1`
- **Edge cases or potential failure types:** none
- **Sub-workflow reference:** none

### Node: Sticky Note3
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:** labels GPT clustering area
- **Input and output connections:** none
- **Version-specific requirements (if any):** type version `1`
- **Edge cases or potential failure types:** none
- **Sub-workflow reference:** none

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Domain Input Form | n8n-nodes-base.formTrigger | Public form entry point for domain/brand/market input |  | Configuration | # Find which AI search topics each domain owns with SE Ranking and GPT<br>## Who is this for<br>- SEO teams wanting to understand topic-level AI search dominance across competitors<br>- Content strategists building editorial plans around AI visibility gaps<br>- Marketing managers benchmarking brand presence across AI search topics<br><br>## What this workflow does<br>Pulls AI search prompts for your domain and up to 2 competitors, then uses GPT to cluster them into topics and reason about which domain owns each one — turning a flat list of prompts into a strategic competitive topic map.<br><br>## What you'll get<br>- AI search leaderboard with share of voice across ChatGPT, Perplexity, Gemini, AI Overviews, and AI Mode<br>- A topic-level competitive map showing which domain wins each topic area<br>- Prompt counts per domain per topic so you can see exactly where you're ahead or behind<br>- A one-line actionable insight per topic to guide your content strategy<br>- An overall winner and competitive summary saved to Google Sheets<br><br>## How it works<br>1. Add your domain and 2 competitors in the form — pulls the AI search leaderboard across all 5 LLM engines<br>2. Fetches up to 10 prompts per domain (both brand and target) for you and each competitor<br>3. Filters competitor prompts to keep only SEO-relevant topics — removes noise like gaming or sports<br>4. Sends all prompts to GPT with instructions to cluster them into topics and identify which domain appears most per topic<br>5. GPT reasons about dominance per cluster and returns a structured competitive topic map<br>6. Saves the leaderboard and topic map to separate tabs in Google Sheets<br><br>## Requirements<br>- SE Ranking community node installed<br>- SE Ranking API token ([Get one here](https://online.seranking.com/admin.api.dashboard.html))<br>- OpenAI API key<br>- Google Sheets account (optional)<br><br>## Setup<br>1. Install the [SE Ranking community node](https://www.npmjs.com/package/@seranking/n8n-nodes-seranking)<br>2. Add your SE Ranking API credentials<br>3. Add your OpenAI API credentials<br>4. Connect your Google Sheets account and set a spreadsheet URL in each export node<br>5. Activate the workflow — n8n generates a unique form URL you can share or embed<br>6. Open the form, fill in your domain and competitors, and the workflow runs automatically<br><br>## Customization<br>- Change `prompts_limit` in the Configuration node to fetch more or fewer prompts per domain<br>- Change `source` in the Configuration node for a different regional database (us, uk, de, fr, es, etc.)<br>- Edit the system prompt in the GPT node to adjust how topics are clustered or how insights are written<br>### 📋 How to share or embed this form<br>1. **Activate** the workflow using the toggle in the top right<br>2. Open the **Domain Input Form** node and copy the **Production URL**<br>3. **Share the link** directly — anyone with the URL can fill in their domain and trigger the workflow<br>4. **Embed on your website** using an iframe:<br>`<iframe src="YOUR_PRODUCTION_URL" width="600" height="500" frameborder="0"></iframe>`<br>Each form submission runs the full workflow automatically. |
| Configuration | n8n-nodes-base.set | Normalizes form values into internal config fields | Domain Input Form | Wait (3s), Wait (6s), Wait (9s), Wait (12s), Wait (15s), Wait (18s) | # Find which AI search topics each domain owns with SE Ranking and GPT<br>## Who is this for<br>- SEO teams wanting to understand topic-level AI search dominance across competitors<br>- Content strategists building editorial plans around AI visibility gaps<br>- Marketing managers benchmarking brand presence across AI search topics<br><br>## What this workflow does<br>Pulls AI search prompts for your domain and up to 2 competitors, then uses GPT to cluster them into topics and reason about which domain owns each one — turning a flat list of prompts into a strategic competitive topic map.<br><br>## What you'll get<br>- AI search leaderboard with share of voice across ChatGPT, Perplexity, Gemini, AI Overviews, and AI Mode<br>- A topic-level competitive map showing which domain wins each topic area<br>- Prompt counts per domain per topic so you can see exactly where you're ahead or behind<br>- A one-line actionable insight per topic to guide your content strategy<br>- An overall winner and competitive summary saved to Google Sheets<br><br>## How it works<br>1. Add your domain and 2 competitors in the form — pulls the AI search leaderboard across all 5 LLM engines<br>2. Fetches up to 10 prompts per domain (both brand and target) for you and each competitor<br>3. Filters competitor prompts to keep only SEO-relevant topics — removes noise like gaming or sports<br>4. Sends all prompts to GPT with instructions to cluster them into topics and identify which domain appears most per topic<br>5. GPT reasons about dominance per cluster and returns a structured competitive topic map<br>6. Saves the leaderboard and topic map to separate tabs in Google Sheets<br><br>## Requirements<br>- SE Ranking community node installed<br>- SE Ranking API token ([Get one here](https://online.seranking.com/admin.api.dashboard.html))<br>- OpenAI API key<br>- Google Sheets account (optional)<br><br>## Setup<br>1. Install the [SE Ranking community node](https://www.npmjs.com/package/@seranking/n8n-nodes-seranking)<br>2. Add your SE Ranking API credentials<br>3. Add your OpenAI API credentials<br>4. Connect your Google Sheets account and set a spreadsheet URL in each export node<br>5. Activate the workflow — n8n generates a unique form URL you can share or embed<br>6. Open the form, fill in your domain and competitors, and the workflow runs automatically<br><br>## Customization<br>- Change `prompts_limit` in the Configuration node to fetch more or fewer prompts per domain<br>- Change `source` in the Configuration node for a different regional database (us, uk, de, fr, es, etc.)<br>- Edit the system prompt in the GPT node to adjust how topics are clustered or how insights are written |
| Wait (3s) | n8n-nodes-base.wait | Delay before first SE Ranking request | Configuration | Get your target prompts | ## Pulls AI search prompts for your domain target and brand |
| Get your target prompts | @seranking/n8n-nodes-seranking.seRanking | Fetch AI-search target prompts for primary domain | Wait (3s) | Merge all prompts | ## Pulls AI search prompts for your domain target and brand |
| Wait (6s) | n8n-nodes-base.wait | Delay before primary brand prompt request | Configuration | Get your brand prompts | ## Pulls AI search prompts for your domain target and brand |
| Get your brand prompts | @seranking/n8n-nodes-seranking.seRanking | Fetch AI-search brand prompts for primary brand | Wait (6s) | Merge all prompts | ## Pulls AI search prompts for your domain target and brand |
| Wait (9s) | n8n-nodes-base.wait | Delay before competitor 1 target request | Configuration | Get competitor 1 target prompts | ## Pulls AI search prompts for your 2 competitors domain target and brand |
| Get competitor 1 target prompts | @seranking/n8n-nodes-seranking.seRanking | Fetch AI-search target prompts for competitor 1 | Wait (9s) | Merge all prompts | ## Pulls AI search prompts for your 2 competitors domain target and brand |
| Wait (12s) | n8n-nodes-base.wait | Delay before competitor 1 brand request | Configuration | Get competitor 1 brand prompts | ## Pulls AI search prompts for your 2 competitors domain target and brand |
| Get competitor 1 brand prompts | @seranking/n8n-nodes-seranking.seRanking | Fetch AI-search brand prompts for competitor 1 | Wait (12s) | Merge all prompts | ## Pulls AI search prompts for your 2 competitors domain target and brand |
| Wait (15s) | n8n-nodes-base.wait | Delay before competitor 2 target request | Configuration | Get competitor 2 target prompts | ## Pulls AI search prompts for your 2 competitors domain target and brand |
| Get competitor 2 target prompts | @seranking/n8n-nodes-seranking.seRanking | Fetch AI-search target prompts for competitor 2 | Wait (15s) | Merge all prompts | ## Pulls AI search prompts for your 2 competitors domain target and brand |
| Wait (18s) | n8n-nodes-base.wait | Delay before competitor 2 brand request | Configuration | Get competitor 2 brand prompts | ## Pulls AI search prompts for your 2 competitors domain target and brand |
| Get competitor 2 brand prompts | @seranking/n8n-nodes-seranking.seRanking | Fetch AI-search brand prompts for competitor 2 | Wait (18s) | Merge all prompts | ## Pulls AI search prompts for your 2 competitors domain target and brand |
| Merge all prompts | n8n-nodes-base.merge | Combine six prompt datasets into one stream | Get your target prompts, Get your brand prompts, Get competitor 1 target prompts, Get competitor 1 brand prompts, Get competitor 2 target prompts, Get competitor 2 brand prompts | Format prompts for GPT |  |
| Format prompts for GPT | n8n-nodes-base.code | Extract, label, filter, and serialize prompt lists for GPT | Merge all prompts | GPT topic clustering | ## GPT cluster prompts into topics and reason about which domain owns each one — turning a flat list of prompts into a strategic competitive topic map. |
| GPT topic clustering | @n8n/n8n-nodes-langchain.openAi | Use OpenAI to cluster prompts and infer topic winners | Format prompts for GPT | Format GPT output | ## GPT cluster prompts into topics and reason about which domain owns each one — turning a flat list of prompts into a strategic competitive topic map. |
| Format GPT output | n8n-nodes-base.code | Parse GPT JSON output and flatten it into sheet rows | GPT topic clustering | Export to Sheets: Topic Analysis | ## GPT cluster prompts into topics and reason about which domain owns each one — turning a flat list of prompts into a strategic competitive topic map. |
| Export to Sheets: Topic Analysis | n8n-nodes-base.googleSheets | Append final topic analysis rows to Google Sheets | Format GPT output |  |  |
| Sticky Note1 | n8n-nodes-base.stickyNote | In-canvas documentation and setup notes |  |  |  |
| Sticky Note | n8n-nodes-base.stickyNote | In-canvas label for competitor collection block |  |  |  |
| Sticky Note4 | n8n-nodes-base.stickyNote | In-canvas form sharing and embed instructions |  |  |  |
| Sticky Note2 | n8n-nodes-base.stickyNote | In-canvas label for primary-domain collection block |  |  |  |
| Sticky Note3 | n8n-nodes-base.stickyNote | In-canvas label for GPT clustering block |  |  |  |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.

2. **Add a Form Trigger node** named `Domain Input Form`.
   - Type: `Form Trigger`
   - Title: `AI Search Topic Analysis`
   - Description: `Enter your domain and up to 2 competitors to see which topics each domain owns in AI search.`
   - Add form fields:
     1. Text: `Your Domain` (required), placeholder `example.com`
     2. Text: `Your Brand Name` (required), placeholder `Example`
     3. Text: `Competitor 1 Domain` (required), placeholder `competitor1.com`
     4. Text: `Competitor 1 Brand` (required), placeholder `Competitor One`
     5. Text: `Competitor 2 Domain` (optional), placeholder `competitor2.com`
     6. Text: `Competitor 2 Brand` (optional), placeholder `Competitor Two`
     7. Dropdown: `Target Market` (required) with options:
        - us
        - uk
        - de
        - fr
        - es
        - it
        - au
        - ca
        - pl

3. **Add a Set node** named `Configuration`.
   - Connect `Domain Input Form` → `Configuration`
   - Add fields:
     - `your_domain` = `{{ $json['Your Domain'] }}`
     - `your_brand` = `{{ $json['Your Brand Name'] }}`
     - `competitor_domain_1` = `{{ $json['Competitor 1 Domain'] }}`
     - `competitor_brand_1` = `{{ $json['Competitor 1 Brand'] }}`
     - `competitor_domain_2` = `{{ $json['Competitor 2 Domain'] || $json['Competitor 1 Domain'] }}`
     - `competitor_brand_2` = `{{ $json['Competitor 2 Brand'] || $json['Competitor 1 Brand'] }}`
     - `source` = `{{ $json['Target Market'] }}`
     - `scope` = `base_domain`
     - `prompts_limit` = `10`

4. **Install the SE Ranking community node** in your n8n instance if not already installed.
   - Package: `@seranking/n8n-nodes-seranking`

5. **Create SE Ranking credentials**.
   - Credential type: SE Ranking API
   - Use your API token from: `https://online.seranking.com/admin.api.dashboard.html`

6. **Create OpenAI credentials**.
   - Credential type: OpenAI API
   - Add your API key
   - Make sure your account has access to the chosen model or replace the model with one available to your account

7. **Create Google Sheets OAuth2 credentials**.
   - Credential type: Google Sheets OAuth2
   - Authorize access to the target spreadsheet

8. **Add six Wait nodes** and connect all of them from `Configuration`.
   - `Wait (3s)` amount = 3
   - `Wait (6s)` amount = 6
   - `Wait (9s)` amount = 9
   - `Wait (12s)` amount = 12
   - `Wait (15s)` amount = 15
   - `Wait (18s)` amount = 18

9. **Add an SE Ranking node** named `Get your target prompts`.
   - Connect `Wait (3s)` → `Get your target prompts`
   - Resource: `aiSearch`
   - Operation: `getPromptsByTarget`
   - Engine: `ai-overview`
   - Scope: `{{ $('Configuration').item.json.scope }}`
   - Domain: `{{ $('Configuration').item.json.your_domain }}`
   - Source: `{{ $('Configuration').item.json.source }}`
   - Additional fields:
     - Sort: `volume`
     - Limit: `{{ $('Configuration').item.json.prompts_limit }}`
     - Sort order: `desc`
   - Assign SE Ranking credentials

10. **Add an SE Ranking node** named `Get your brand prompts`.
    - Connect `Wait (6s)` → `Get your brand prompts`
    - Resource: `aiSearch`
    - Operation: `getPromptsByBrand`
    - Engine: `ai-overview`
    - Brand name: `{{ $('Configuration').item.json.your_brand }}`
    - Source: `{{ $('Configuration').item.json.source }}`
    - Additional fields:
      - Sort: `volume`
      - Limit: `{{ $('Configuration').item.json.prompts_limit }}`
      - Sort order: `desc`
    - Assign SE Ranking credentials

11. **Add an SE Ranking node** named `Get competitor 1 target prompts`.
    - Connect `Wait (9s)` → `Get competitor 1 target prompts`
    - Resource: `aiSearch`
    - Operation: `getPromptsByTarget`
    - Engine: `ai-overview`
    - Scope: `{{ $('Configuration').item.json.scope }}`
    - Domain: `{{ $('Configuration').item.json.competitor_domain_1 }}`
    - Source: `{{ $('Configuration').item.json.source }}`
    - Additional fields:
      - Sort: `volume`
      - Limit: `{{ $('Configuration').item.json.prompts_limit }}`
      - Sort order: `desc`
    - Assign SE Ranking credentials

12. **Add an SE Ranking node** named `Get competitor 1 brand prompts`.
    - Connect `Wait (12s)` → `Get competitor 1 brand prompts`
    - Resource: `aiSearch`
    - Operation: `getPromptsByBrand`
    - Engine: `ai-overview`
    - Brand name: `{{ $('Configuration').item.json.competitor_brand_1 }}`
    - Source: `{{ $('Configuration').item.json.source }}`
    - Additional fields:
      - Sort: `volume`
      - Limit: `{{ $('Configuration').item.json.prompts_limit }}`
      - Sort order: `desc`
    - Assign SE Ranking credentials

13. **Add an SE Ranking node** named `Get competitor 2 target prompts`.
    - Connect `Wait (15s)` → `Get competitor 2 target prompts`
    - Resource: `aiSearch`
    - Operation: `getPromptsByTarget`
    - Engine: `ai-overview`
    - Scope: `{{ $('Configuration').item.json.scope }}`
    - Domain: `{{ $('Configuration').item.json.competitor_domain_2 }}`
    - Source: `{{ $('Configuration').item.json.source }}`
    - Additional fields:
      - Sort: `volume`
      - Limit: `{{ $('Configuration').item.json.prompts_limit }}`
      - Sort order: `desc`
    - Assign SE Ranking credentials

14. **Add an SE Ranking node** named `Get competitor 2 brand prompts`.
    - Connect `Wait (18s)` → `Get competitor 2 brand prompts`
    - Resource: `aiSearch`
    - Operation: `getPromptsByBrand`
    - Engine: `ai-overview`
    - Brand name: `{{ $('Configuration').item.json.competitor_brand_2 }}`
    - Source: `{{ $('Configuration').item.json.source }}`
    - Additional fields:
      - Sort: `volume`
      - Limit: `{{ $('Configuration').item.json.prompts_limit }}`
      - Sort order: `desc`
    - Assign SE Ranking credentials

15. **Add a Merge node** named `Merge all prompts`.
    - Set number of inputs to `6`
    - Connect:
      - `Get your target prompts` → input 0
      - `Get your brand prompts` → input 1
      - `Get competitor 1 target prompts` → input 2
      - `Get competitor 1 brand prompts` → input 3
      - `Get competitor 2 target prompts` → input 4
      - `Get competitor 2 brand prompts` → input 5

16. **Add a Code node** named `Format prompts for GPT`.
    - Connect `Merge all prompts` → `Format prompts for GPT`
    - Paste this logic conceptually:
      - collect all merged items
      - get config values from `Configuration`
      - define an SEO keyword list
      - map each merged result to a labeled domain section
      - extract `prompt` values from each response’s `prompts` array
      - filter competitor prompts to SEO-related terms only
      - join all sections into a single string called `prompt_data`
      - return one item containing `prompt_data` plus all domain/brand variables
    - Use the same field behavior as the provided workflow if you want exact replication

17. **Add an OpenAI node** named `GPT topic clustering`.
    - Connect `Format prompts for GPT` → `GPT topic clustering`
    - Use the OpenAI/LangChain node
    - Model: `gpt-5.4-nano-2026-03-17` or another available lightweight model
    - Temperature: `0.3`
    - Add a user message:
      - `{{ $json.prompt_data }}`
    - Add a system message instructing the model to:
      - cluster prompts into 5–10 meaningful SEO topic groups
      - count prompts per domain
      - identify topic winner
      - provide one actionable insight per topic
      - include representative prompts from all domains
      - return only valid JSON in this structure:
        - `topics` array
        - `overall_winner`
        - `summary`

18. **Add a Code node** named `Format GPT output`.
    - Connect `GPT topic clustering` → `Format GPT output`
    - Implement logic to:
      - detect text content from possible OpenAI response formats
      - remove code fences if present
      - parse JSON
      - if parsing fails, return one item with an `error` and raw response
      - otherwise map each topic into one row with:
        - `topic`
        - `winner`
        - `<your_domain>_prompts`
        - `<competitor_domain_1>_prompts`
        - `<competitor_domain_2>_prompts`
        - `insight`
        - `top_prompts`
        - `overall_winner`
        - `summary`
        - `date`

19. **Add a Google Sheets node** named `Export to Sheets: Topic Analysis`.
    - Connect `Format GPT output` → `Export to Sheets: Topic Analysis`
    - Operation: `Append`
    - Choose your spreadsheet
    - Set the target sheet/tab; the original workflow uses `gid=0`
    - Assign Google Sheets OAuth2 credentials

20. **Test the workflow manually**.
    - Submit the form
    - Confirm SE Ranking nodes return prompt arrays
    - Confirm `Format prompts for GPT` generates a non-empty `prompt_data`
    - Confirm GPT returns parseable JSON
    - Confirm rows are appended to Sheets

21. **Activate the workflow** to enable the production form URL.

22. **Share or embed the form** if needed.
    - Copy the Form Trigger production URL
    - Optionally embed with:
      ```html
      <iframe
        src="YOUR_PRODUCTION_URL"
        width="600"
        height="500"
        frameborder="0">
      </iframe>
      ```

### Important implementation notes when rebuilding
1. **Competitor 2 fallback behavior**
   - The workflow intentionally duplicates competitor 1 into competitor 2 if the second competitor is omitted.
   - If you do not want duplicated comparisons, replace fallback logic with blank-safe branching.

2. **Google Sheets schema**
   - Because the output uses dynamic column names based on domains, different executions may create inconsistent headers.
   - A more stable design would use fixed columns like `your_domain_count`, `competitor_1_count`, `competitor_2_count`.

3. **SE Ranking response assumptions**
   - The Code node expects a structure resembling:
     - one item
     - `prompts` array
     - each element has a `prompt` property

4. **Rate limiting**
   - The wait nodes are part of the logic, not decorative.
   - If you increase API volume, extend delays or add retry/error handling.

5. **Model availability**
   - If `gpt-5.4-nano-2026-03-17` is unavailable, choose another JSON-capable model and keep the system prompt strict.

6. **No sub-workflows are used**
   - This workflow has one entry point only: the form trigger.
   - No Execute Workflow or sub-workflow nodes are present.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| SE Ranking API token can be created from the SE Ranking API dashboard | https://online.seranking.com/admin.api.dashboard.html |
| SE Ranking community node package required for this workflow | https://www.npmjs.com/package/@seranking/n8n-nodes-seranking |
| The form works only after the workflow is activated and a production URL is generated | Applies to `Domain Input Form` |
| The form can be embedded in a website using an iframe | Use the Form Trigger production URL |
| Workflow positioning notes indicate three visual areas: primary-domain prompt collection, competitor prompt collection, and GPT clustering/output | Canvas organization |
| The in-canvas overview claims the workflow saves both a leaderboard and a topic map to separate tabs, but the provided JSON only includes one Google Sheets export node for topic analysis | Functional discrepancy to note before deployment |