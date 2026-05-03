Qwen-Max: Journal Paper Generation from Title/Abstract

https://n8nworkflows.xyz/workflows/qwen-max--journal-paper-generation-from-title-abstract-10314


# Qwen-Max: Journal Paper Generation from Title/Abstract

---

### 1. Workflow Overview

This workflow, titled **"AI-Powered Qwen-Max Journal Paper Generator from Title and Abstract"**, automates the generation of a complete academic journal paper based on a user-provided paper title and abstract. It is designed for researchers, academics, and anyone needing to draft detailed scientific papers efficiently.

The workflow is structured into three main logical blocks:

- **1.1 Input Reception and Reference Gathering:** Receives user input (paper title and abstract) via a webhook, queries multiple academic databases (CrossRef, Semantic Scholar, OpenAlex) for relevant literature, merges these sources, and processes references to build a clean, deduplicated bibliography.

- **1.2 AI-Powered Section Generation:** Prepares the AI context with input data and references, then uses the Qwen-Max language model via OpenRouter to generate six core sections of the paper: Introduction, Literature Review, Methodology, Results, Discussion, and Conclusion. Each section incorporates citations in APA format with academic rigor and originality.

- **1.3 Document Assembly:** Combines the generated sections and input data into a fully formatted academic paper with a formatted reference list, ready for export or further use.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Reference Gathering

**Overview:**  
This block ingests the initial paper title and abstract from an external POST request, then performs parallel searches across three major academic databases to collect relevant references. It merges and processes these references to create a cleaned, deduplicated set of citations.

**Nodes Involved:**  
- Webhook  
- Extract Input Data  
- Search CrossRef  
- Search Semantic Scholar  
- Search OpenAlex  
- Merge Reference Sources  
- Process References  

**Node Details:**

- **Webhook**  
  - *Type:* Webhook (HTTP POST trigger)  
  - *Configuration:* Listens on path `/generate-paper`, accepts POST requests, uses response mode `responseNode` to send back responses downstream.  
  - *Connections:* Output → Extract Input Data  
  - *Edge cases:* Invalid HTTP method or malformed POST body; missing required fields (`title`, `abstract`).

- **Extract Input Data**  
  - *Type:* Set node  
  - *Role:* Extracts and assigns `paperTitle` and `paperAbstract` from the webhook request JSON body.  
  - *Key variables:* `paperTitle = {{$json.body.title}}`, `paperAbstract = {{$json.body.abstract}}`  
  - *Input:* Webhook output JSON  
  - *Output:* JSON with extracted title and abstract  
  - *Edge cases:* Missing or empty title/abstract fields.

- **Search CrossRef**  
  - *Type:* HTTP Request  
  - *Role:* Searches CrossRef API for papers relevant to `paperTitle`.  
  - *Configuration:*  
    - URL: `https://api.crossref.org/works`  
    - Query params: `query.title` = paper title, `rows` = 12 (limit results)  
    - Authentication: Generic HTTP Query Auth (configured credentials)  
  - *Input:* Extracted input data  
  - *Output:* JSON with CrossRef results  
  - *Edge cases:* API rate limits, network errors, invalid API keys.

- **Search Semantic Scholar**  
  - *Type:* HTTP Request  
  - *Role:* Queries Semantic Scholar API for papers matching the paper title.  
  - *Configuration:*  
    - URL: `https://api.semanticscholar.org/graph/v1/paper/search`  
    - Query params:  
      - `query` = paper title  
      - `limit` = 12  
      - `fields` = title, authors, year, abstract, citationCount, url, venue, publicationDate  
  - *Input:* Extracted input data  
  - *Output:* JSON with Semantic Scholar results  
  - *Edge cases:* API throttling, missing fields, network errors.

- **Search OpenAlex**  
  - *Type:* HTTP Request  
  - *Role:* Queries OpenAlex API for papers relevant to the paper title.  
  - *Configuration:*  
    - URL: `https://api.openalex.org/works`  
    - Query params:  
      - `search` = paper title  
      - `per-page` = 11  
  - *Input:* Extracted input data  
  - *Output:* JSON with OpenAlex results  
  - *Edge cases:* API limits, malformed responses.

- **Merge Reference Sources**  
  - *Type:* Merge node  
  - *Role:* Combines results from the three searches into a single data stream.  
  - *Configuration:* Mode `combine` (concatenate all inputs)  
  - *Input:* Outputs from Search CrossRef, Semantic Scholar, OpenAlex  
  - *Output:* Combined reference data array  
  - *Edge cases:* Different JSON structures; empty results.

- **Process References**  
  - *Type:* Code node (JavaScript)  
  - *Role:* Parses and normalizes references from all three sources into a uniform format, removes duplicates based on title similarity, and limits to top 35 references.  
  - *Key logic:*  
    - Maps fields (title, authors, year, venue, DOI, URL, abstract) from each source's schema.  
    - Uses a Set to track normalized titles to avoid duplicates.  
  - *Input:* Merged references  
  - *Output:* JSON object with cleaned `references` array  
  - *Edge cases:* Missing fields, inconsistent data formats, empty inputs.

---

#### 2.2 AI-Powered Section Generation

**Overview:**  
This block prepares the AI input context with paper data and references, then sequentially generates all major paper sections using the Qwen-Max model via OpenRouter API integration. Each section has a dedicated prompt specifying academic writing requirements, citation style, and length.

**Nodes Involved:**  
- Prepare AI Context  
- OpenRouter Chat Model  
- AI - Introduction  
- AI - Literature Review  
- AI - Methodology  
- AI - Results  
- AI - Discussion  
- AI - Conclusion  
- Merge All Sections  

**Node Details:**

- **Prepare AI Context**  
  - *Type:* Set node  
  - *Role:* Creates the AI input context by packaging `paperTitle`, `paperAbstract`, and processed `references` into one JSON payload for AI nodes.  
  - *Variables:*  
    - `paperTitle` from Extract Input Data  
    - `paperAbstract` from Extract Input Data  
    - `references` from Process References  
  - *Input:* Process References output  
  - *Output:* JSON with prepared context  
  - *Edge cases:* Empty reference list.

- **OpenRouter Chat Model**  
  - *Type:* AI language model node (Qwen-Max via OpenRouter)  
  - *Role:* Provides the large language model backend for text generation across AI section nodes.  
  - *Configuration:* Model set to `qwen/qwen-max`, credentials attached for OpenRouter API.  
  - *Input:* Receives prompts and context from AI section nodes.  
  - *Output:* Generated text content for each paper section.  
  - *Edge cases:* API key errors, quota exceeded, timeouts.

- **AI - Introduction**  
  - *Type:* LangChain Agent node  
  - *Role:* Generates the paper’s Introduction section using the prepared context and references.  
  - *Prompt focus:* Background, gap, objectives, structure, APA citations, 800-1000 words.  
  - *Input:* Prepare AI Context output  
  - *Output:* Introduction text with citations  
  - *Connections:* Output feeds AI - Literature Review and Merge All Sections nodes.  
  - *Edge cases:* Incomplete context, insufficient references.

- **AI - Literature Review**  
  - *Type:* LangChain Agent node  
  - *Role:* Creates an extensive Literature Review section synthesizing references with 15-20 citations.  
  - *Prompt focus:* Thematic synthesis, trends, gaps, logical organization, 1500-2000 words.  
  - *Input:* Output of AI - Introduction  
  - *Output:* Literature Review text  
  - *Connections:* Output feeds AI - Methodology and Merge All Sections (second output path).  
  - *Edge cases:* Overlapping content with Introduction, insufficient references.

- **AI - Methodology**  
  - *Type:* LangChain Agent node  
  - *Role:* Drafts detailed Methodology based on title and abstract, including design, data collection, analysis, validity, citations.  
  - *Prompt focus:* 1000-1200 words, coherent, replicable, APA citations.  
  - *Input:* Output of AI - Literature Review  
  - *Output:* Methodology text  
  - *Connections:* Output feeds AI - Results  
  - *Edge cases:* Ambiguous input data, generic methodology.

- **AI - Results**  
  - *Type:* LangChain Agent node  
  - *Role:* Generates Results section with plausible findings and objective presentation.  
  - *Prompt focus:* 1000-1200 words, conceptual data presentation, no interpretation.  
  - *Input:* Output of AI - Methodology  
  - *Output:* Results text  
  - *Connections:* Output feeds AI - Discussion  
  - *Edge cases:* Lack of real data, generic results.

- **AI - Discussion**  
  - *Type:* LangChain Agent node  
  - *Role:* Produces Discussion interpreting results, comparing literature, addressing hypotheses, implications, limitations.  
  - *Prompt focus:* 1200-1500 words, deep analysis, extensive citations.  
  - *Input:* Output of AI - Results  
  - *Output:* Discussion text  
  - *Connections:* Output feeds AI - Conclusion  
  - *Edge cases:* Overinterpretation, unsupported claims.

- **AI - Conclusion**  
  - *Type:* LangChain Agent node  
  - *Role:* Creates conclusion summarizing findings, contributions, implications, future work, closure.  
  - *Prompt focus:* 600-800 words, minimal citations.  
  - *Input:* Output of AI - Discussion  
  - *Output:* Conclusion text  
  - *Edge cases:* Repetition, lack of closure.

- **Merge All Sections**  
  - *Type:* Merge node  
  - *Role:* Combines the outputs of Introduction, Literature Review, Methodology, Results, Discussion, Conclusion into a single bundle for compilation.  
  - *Mode:* Combine  
  - *Input:* Outputs from AI - Introduction (first), AI - Literature Review (second)  
  - *Output:* Combined sections  
  - *Edge cases:* Missing sections due to failures in AI nodes.

---

#### 2.3 Document Assembly

**Overview:**  
Finalizes the workflow by compiling all generated sections and references into a fully formatted academic paper document, including APA-style reference formatting and word count.

**Nodes Involved:**  
- Compile Document  

**Node Details:**

- **Compile Document**  
  - *Type:* Code node (JavaScript)  
  - *Role:*  
    - Retrieves paper title, abstract, and AI-generated sections from previous nodes.  
    - Formats references into APA style strings.  
    - Concatenates all parts into a structured document with headings for each section and the reference list.  
    - Calculates total word count for the full document.  
  - *Input:* Merged sections and Extract Input Data, Process References outputs  
  - *Output:* JSON containing:  
    - `fullDocument` (string with entire paper)  
    - `title`, `abstract`, each section’s text, formatted references, and word count  
  - *Edge cases:* Missing or incomplete section texts, formatting inconsistencies.

---

### 3. Summary Table

| Node Name              | Node Type                          | Functional Role                                | Input Node(s)                           | Output Node(s)                             | Sticky Note                                                                                                               |
|------------------------|----------------------------------|-----------------------------------------------|---------------------------------------|--------------------------------------------|---------------------------------------------------------------------------------------------------------------------------|
| Webhook                | Webhook                          | Input reception of paper title and abstract  | -                                     | Extract Input Data                         |                                                                                                                           |
| Extract Input Data      | Set                              | Extracts title and abstract from webhook input| Webhook                               | Search CrossRef, Search Semantic Scholar, Search OpenAlex |                                                                                                                           |
| Search CrossRef         | HTTP Request                    | Searches CrossRef database for references     | Extract Input Data                    | Merge Reference Sources                    |                                                                                                                           |
| Search Semantic Scholar | HTTP Request                    | Searches Semantic Scholar for references      | Extract Input Data                    | Merge Reference Sources                    |                                                                                                                           |
| Search OpenAlex         | HTTP Request                    | Searches OpenAlex database for references     | Extract Input Data                    | Merge Reference Sources                    |                                                                                                                           |
| Merge Reference Sources | Merge                            | Combines reference results from all sources   | Search CrossRef, Search Semantic Scholar, Search OpenAlex | Process References                         |                                                                                                                           |
| Process References      | Code                             | Normalizes and deduplicates references         | Merge Reference Sources               | Prepare AI Context                         |                                                                                                                           |
| Prepare AI Context      | Set                              | Packages title, abstract, references for AI   | Process References                   | AI - Introduction                          |                                                                                                                           |
| OpenRouter Chat Model   | AI Language Model                | Provides Qwen-Max model for AI generation      | Various AI section nodes             | AI section nodes (Introduction to Conclusion) |                                                                                                                           |
| AI - Introduction      | LangChain Agent                  | Generates Introduction section                  | Prepare AI Context, OpenRouter Chat Model | AI - Literature Review, Merge All Sections |                                                                                                                           |
| AI - Literature Review | LangChain Agent                  | Generates Literature Review section             | AI - Introduction                   | AI - Methodology, Merge All Sections       |                                                                                                                           |
| AI - Methodology       | LangChain Agent                  | Generates Methodology section                    | AI - Literature Review              | AI - Results                               |                                                                                                                           |
| AI - Results           | LangChain Agent                  | Generates Results section                        | AI - Methodology                   | AI - Discussion                            |                                                                                                                           |
| AI - Discussion        | LangChain Agent                  | Generates Discussion section                     | AI - Results                      | AI - Conclusion                            |                                                                                                                           |
| AI - Conclusion        | LangChain Agent                  | Generates Conclusion section                     | AI - Discussion                   | -                                          |                                                                                                                           |
| Merge All Sections     | Merge                            | Combines all AI-generated sections              | AI - Introduction, AI - Literature Review | Compile Document                           |                                                                                                                           |
| Compile Document       | Code                             | Compiles full paper text and formats references| Merge All Sections, Extract Input Data, Process References | -                                          |                                                                                                                           |
| Sticky Note            | Sticky Note                     | Provides workflow overview and instructions    | -                                   | -                                          | ## Introduction\nGenerates complete scientific papers from title and abstract using AI. Designed for researchers, automating literature search, content generation, and citation formatting.\n## How It Works\nExtracts input, searches academic databases (CrossRef, Semantic Scholar, OpenAlex), merges sources, processes citations, generates AI sections (Introduction, Literature Review, Methodology, Results, Discussion, Conclusion), compiles document.\n## Workflow Template\nWebhook → Extract Data → Search (CrossRef + Semantic Scholar + OpenAlex) → Merge Sources → Process References → Prepare Context → AI Generate (Introduction + Literature Review + Methodology + Results + Discussion + Conclusion via OpenAI) → Merge Sections → Compile Document\n## Workflow Steps\n1. **Input & Search:** Webhook receives title/abstract; searches CrossRef, Semantic Scholar, OpenAlex; merges and processes references\n2. **AI Generation:** OpenAI generates six sections with in-text citations using retrieved references\n3. **Assembly:** Merges sections; compiles formatted document with reference list |
| Sticky Note1           | Sticky Note                     | Setup instructions and benefits                 | -                                   | -                                          | ## Setup Instructions\n1. **Trigger & APIs:** Configure webhook URL; add OpenAI API key; customize prompts\n2. **Databases:** Set up CrossRef, Semantic Scholar, OpenAlex API access; configure search parameters\n## Prerequisites\nOpenAI API, CrossRef API, Semantic Scholar API, OpenAlex API, webhook platform, n8n instance\n## Customization\nAdjust reference limits, modify prompts for research fields, add citation styles (APA/IEEE)\n## Benefits\nAutomates paper drafting, comprehensive literature integration, proper citations |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node:**  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: `generate-paper`  
   - Response Mode: `responseNode`  
   - Purpose: Receive JSON input with `title` and `abstract`.

2. **Add Set Node - Extract Input Data:**  
   - Create variables:  
     - `paperTitle` = `{{$json.body.title}}`  
     - `paperAbstract` = `{{$json.body.abstract}}`  
   - Connect Webhook output to this node.

3. **Add HTTP Request Nodes to Query Databases:**  
   - **CrossRef:**  
     - URL: `https://api.crossref.org/works`  
     - Query Params: `query.title` = `{{$json.paperTitle}}`; `rows` = 12  
     - Authentication: Configure generic HTTP Query Auth with CrossRef credentials  
     - Connect Extract Input Data output to this node.  
   - **Semantic Scholar:**  
     - URL: `https://api.semanticscholar.org/graph/v1/paper/search`  
     - Query Params:  
       - `query` = `{{$('Extract Input Data').item.json.paperTitle}}`  
       - `limit` = 12  
       - `fields` = `title,authors,year,abstract,citationCount,url,venue,publicationDate`  
     - Connect Extract Input Data output to this node.  
   - **OpenAlex:**  
     - URL: `https://api.openalex.org/works`  
     - Query Params:  
       - `search` = `{{$('Extract Input Data').item.json.paperTitle}}`  
       - `per-page` = 11  
     - Connect Extract Input Data output to this node.

4. **Add Merge Node - Merge Reference Sources:**  
   - Mode: Combine  
   - Connect outputs of the three search nodes to this merge node.

5. **Add Code Node - Process References:**  
   - JavaScript code to:  
     - Extract relevant fields from each source’s JSON response.  
     - Normalize author names, years, titles, venues, DOIs, abstracts.  
     - Remove duplicates by title (case-insensitive).  
     - Limit to top 35 references.  
   - Connect Merge Reference Sources output to this node.

6. **Add Set Node - Prepare AI Context:**  
   - Assign variables:  
     - `paperTitle` from Extract Input Data output  
     - `paperAbstract` from Extract Input Data output  
     - `references` from Process References output  
   - Connect Process References output here.

7. **Configure OpenRouter Chat Model Node:**  
   - Type: AI Language Model (Qwen-Max via OpenRouter)  
   - Model: `qwen/qwen-max`  
   - Credentials: Set OpenRouter API credentials  
   - This node will serve as the language model for all AI generation nodes.

8. **Add LangChain Agent Nodes for AI Sections:**  
   - Create six nodes, one for each section: Introduction, Literature Review, Methodology, Results, Discussion, Conclusion.  
   - Configure each with detailed system prompts specifying academic writing requirements, word counts, citation style (APA), and section-specific instructions as per the workflow.  
   - Connect Prepare AI Context to AI - Introduction node.  
   - Chain section outputs sequentially:  
     - Introduction → Literature Review → Methodology → Results → Discussion → Conclusion.  
   - Configure each AI node to use the OpenRouter Chat Model node as its language model.

9. **Add Merge Node - Merge All Sections:**  
   - Mode: Combine  
   - Connect AI - Introduction and AI - Literature Review outputs to this node (both as separate inputs).  
   - Ensure outputs from other AI section nodes feed into this merge as appropriate (or adjust merging to include all six sections).

10. **Add Code Node - Compile Document:**  
    - JavaScript code to:  
      - Retrieve title, abstract, references, and all AI-generated sections.  
      - Format references in APA style.  
      - Concatenate all components into a single formatted document string with section headings.  
      - Calculate total word count.  
    - Connect Merge All Sections output here.

11. **Connect Webhook output to final Compile Document node for response delivery.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                             | Context or Link                                                                                                                  |
|------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------|
| Generates complete scientific papers from title and abstract using AI. Designed for researchers, automating literature search, content generation, and citation formatting. | Sticky Note overview in workflow describing purpose and high-level steps.                                                       |
| Setup Instructions: Configure webhook URL; add OpenAI API key; customize prompts; set up CrossRef, Semantic Scholar, OpenAlex API access; adjust reference limits and citation styles. | Sticky Note1 describing setup prerequisites and customization options.                                                          |
| Uses APA citation style for in-text citations and final reference list formatting.                                                       | Specified in AI generation prompts and compilation code node.                                                                   |
| Relies on OpenRouter API with Qwen-Max model for AI text generation. Requires valid OpenRouter API credentials.                         | OpenRouter Chat Model node configuration.                                                                                       |
| References are deduplicated based on normalized titles to avoid citation redundancy.                                                    | Logic implemented in Process References code node.                                                                               |

---

**Disclaimer:** The provided description and analysis are based exclusively on an n8n workflow automation. All data and operations respect applicable content policies and handle only lawful, non-sensitive information.

---