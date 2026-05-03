Create Comprehensive Research Reports with Jina AI & Gemini 2.5 Flash

https://n8nworkflows.xyz/workflows/create-comprehensive-research-reports-with-jina-ai---gemini-2-5-flash-5463


# Create Comprehensive Research Reports with Jina AI & Gemini 2.5 Flash

---

## 1. Workflow Overview

**Title:** Create Comprehensive Research Reports with Jina AI & Gemini 2.5 Flash  
**Workflow Name:** AI Research Assistant with Jina & Gemini

This workflow automates the generation of detailed, well-cited research reports based on user chat queries. It combines real-time web searching via Jina AI with advanced AI language models (Google Gemini 2.5 Flash) to summarize gathered content, synthesize research, and refine the final report.

The workflow is logically divided into the following functional blocks:

- **1.1 Input Reception**: Captures user chat input as a research query.
- **1.2 Web Search & URL Extraction**: Uses Jina AI to perform web searches and extract relevant URLs.
- **1.3 Content Retrieval & Summarization Loop**: For each URL, fetches content, summarizes it using Gemini-powered AI.
- **1.4 Summaries Aggregation & Research Report Generation**: Consolidates summaries and generates a comprehensive research report.
- **1.5 Report Evaluation & Refinement**: Evaluates the generated report to ensure citation accuracy, formatting, and quality.

---

## 2. Block-by-Block Analysis

### 2.1 Input Reception

**Overview:**  
Receives chat messages from users containing research queries, triggering the workflow.

**Nodes Involved:**  
- When chat message received  
- Sticky Note (User query)

**Node Details:**

- **When chat message received**  
  - Type: `@n8n/n8n-nodes-langchain.chatTrigger`  
  - Role: Entry point webhook node that listens for chat messages to start the workflow.  
  - Configuration: Uses a webhook ID to receive chat input; no additional options set.  
  - Key Expressions: Extracts user query from `chatInput` JSON field.  
  - Input: External chat message via webhook.  
  - Output: Passes user input downstream to "Search web".  
  - Failure Types: Webhook availability issues, malformed input.  

- **Sticky Note (User query)**  
  - Type: `n8n-nodes-base.stickyNote`  
  - Role: Documentation for user query input.  
  - Content: Labeling this block as "User query" for clarity.  

---

### 2.2 Web Search & URL Extraction

**Overview:**  
Performs a web search based on the user query using Jina AI, then extracts URLs from the search results.

**Nodes Involved:**  
- Search web  
- Code (URL extraction)  
- Sticky Note1 (Search web)  
- Sticky Note2 (Code URL Transform)

**Node Details:**

- **Search web**  
  - Type: `n8n-nodes-base.jinaAi`  
  - Role: Executes a web search query using Jina AI API.  
  - Configuration: Operation set to "search" with query parameter bound dynamically to user input (`{{$json.chatInput}}`).  
  - Credentials: Uses stored Jina AI API key.  
  - Input: User chat query from "When chat message received".  
  - Output: Search result items containing metadata including URLs.  
  - Failure Types: API auth errors, network timeouts, empty/no results.  

- **Code (URL extraction)**  
  - Type: `n8n-nodes-base.code` (Python)  
  - Role: Processes raw search results to extract URLs into a simplified list for further processing.  
  - Configuration: Python script iterates over all input items, collects objects with `url` property into a list.  
  - Input: Search results from "Search web".  
  - Output: List of URL objects (e.g., `[{url: "..."}, ...]`).  
  - Failure Types: Missing URL fields, unexpected data structures.  

- **Sticky Note1 (Search web)**  
  - Purpose: Documents the web search step, emphasizing dependency on Jina AI API key.  

- **Sticky Note2 (Code URL Transform)**  
  - Purpose: Explains the Python code's role in URL extraction and preparation for sequential processing.  

---

### 2.3 Content Retrieval & Summarization Loop

**Overview:**  
Iterates over each extracted URL to fetch web content and generate summarized outputs using AI.

**Nodes Involved:**  
- Loop Over Items  
- Read URL content  
- Summarizer Agent  
- Summarizer Model  
- Wait  
- Structured Output  
- Sticky Note3 (Loop over URLs)  
- Sticky Note4 (Read URL content)  
- Sticky Note5 (Summarize Gathered content)  
- Sticky Note6 (Avoid rate limits)

**Node Details:**

- **Loop Over Items**  
  - Type: `n8n-nodes-base.splitInBatches`  
  - Role: Iterates over each URL extracted, processing one batch (default batch size is 1) at a time.  
  - Configuration: Reset disabled to maintain batch state.  
  - Input: URLs from "Code".  
  - Outputs: Two connections — one for processed batch aggregation, one for sequential single URL processing.  
  - Failure Types: Batch processing failures, empty batches.  

- **Read URL content**  
  - Type: `n8n-nodes-base.jinaAi`  
  - Role: Retrieves full content from the provided URL using Jina AI's content fetching capabilities.  
  - Configuration: URL dynamically set from looped item (`{{$json.url}}`).  
  - Credentials: Uses Jina AI API key.  
  - Input: URL from "Loop Over Items".  
  - Output: Raw web page content JSON.  
  - Failure Types: URL unreachable, HTTP errors, API failures.  

- **Summarizer Agent**  
  - Type: `@n8n/n8n-nodes-langchain.agent`  
  - Role: AI agent that instructs the summarization model to condense fetched content.  
  - Configuration: Prompt dynamically composed including source URL, title, description, content, and initial user query.  
  - Input: Content from "Read URL content".  
  - Output: Summarized content with structured output parser.  
  - Failure Types: Prompt errors, AI model rate limits.  

- **Summarizer Model**  
  - Type: `@n8n/n8n-nodes-langchain.lmChatGoogleGemini`  
  - Role: Google Gemini 2.5 Flash model used for summarization.  
  - Credentials: Requires Google Palm API credentials.  
  - Input: Prompt from "Summarizer Agent".  
  - Output: Summarization text.  
  - Failure Types: API quota exceeded, model unavailability.  

- **Wait**  
  - Type: `n8n-nodes-base.wait`  
  - Role: Pauses workflow for 1 second to prevent API rate limiting.  
  - Configuration: Delay set to 1 second.  
  - Input: After summarization.  
  - Output: Passes to next loop iteration or aggregation.  
  - Failure Types: None typical; misconfiguration may cause delays in processing.  

- **Structured Output**  
  - Type: `@n8n/n8n-nodes-langchain.outputParserStructured`  
  - Role: Parses summarized AI output into a JSON structure with keys `source_url` and `summarized_content`.  
  - Configuration: Example JSON schema provided as template.  
  - Input: From "Summarizer Agent".  
  - Output: Structured summary data for report generation.  
  - Failure Types: Parsing errors if AI output deviates from schema.  

- **Sticky Note3 (Loop over URLs)**  
  - Purpose: Explains the iteration over URLs for individual processing.  

- **Sticky Note4 (Read URL content)**  
  - Purpose: Documents content fetching role and its importance for summarization.  

- **Sticky Note5 (Summarize Gathered content)**  
  - Purpose: Describes AI-powered summarization using Gemini model.  

- **Sticky Note6 (Avoid rate limits)**  
  - Purpose: Highlights the wait node’s role to avoid hitting API limitations.  

---

### 2.4 Summaries Aggregation & Research Report Generation

**Overview:**  
Combines all individual summaries into a consolidated data object and generates a comprehensive research report using an AI agent.

**Nodes Involved:**  
- Transform (Summaries Aggregator)  
- Generator Model  
- Generator Agent  
- Sticky Note7 (Transform Summaries Aggregator)  
- Sticky Note8 (Generate research Report)

**Node Details:**

- **Transform (Summaries Aggregator)**  
  - Type: `n8n-nodes-base.code` (Python)  
  - Role: Aggregates all summarized outputs into a single JSON object with a key `"output"` containing a list of all summaries.  
  - Input: Multiple summarized items from "Loop Over Items".  
  - Output: Single item containing aggregated summaries.  
  - Failure Types: Empty input arrays, malformed summary objects.  

- **Generator Model**  
  - Type: `@n8n/n8n-nodes-langchain.lmChatGoogleGemini`  
  - Role: Google Gemini 2.5 Flash model for generating the full research report based on aggregated summaries.  
  - Credentials: Google Palm API credentials required.  
  - Input: Prompt from "Generator Agent".  
  - Output: Draft research report text.  
  - Failure Types: API quota or model access issues.  

- **Generator Agent**  
  - Type: `@n8n/n8n-nodes-langchain.agent`  
  - Role: Specialized AI agent that synthesizes multiple source summaries into a detailed research report with citations.  
  - Configuration: Complex prompt defining system instructions, input JSON format, research structure requirements, citation guidelines, and quality standards.  
  - Input: Aggregated summaries from "Transform".  
  - Output: Generated research report draft.  
  - Failure Types: Prompt parsing errors, incomplete data from summaries.  

- **Sticky Note7 (Transform Summaries Aggregator)**  
  - Purpose: Documents the aggregation step’s function in consolidating summaries.  

- **Sticky Note8 (Generate research Report)**  
  - Purpose: Details the AI agent’s role in orchestrating research report creation based on summaries.  

---

### 2.5 Report Evaluation & Refinement

**Overview:**  
Processes the draft research report to verify citations, improve formatting, and ensure professional quality.

**Nodes Involved:**  
- Evaluator Model  
- Evaluator Chain  
- Sticky Note9 (Evaluate and Generate Final Report)

**Node Details:**

- **Evaluator Model**  
  - Type: `@n8n/n8n-nodes-langchain.lmChatGoogleGemini`  
  - Role: Google Gemini 2.5 Flash model used to evaluate and refine the research report text.  
  - Credentials: Google Palm API credentials required.  
  - Input: Raw report from "Generator Agent".  
  - Output: Refined, publication-ready research report.  
  - Failure Types: API errors, model limits.  

- **Evaluator Chain**  
  - Type: `@n8n/n8n-nodes-langchain.chainLlm`  
  - Role: AI chain executing a multi-step evaluation process focusing on citation verification, content quality, markdown formatting, and professional presentation.  
  - Configuration: Detailed prompt includes instructions for citation correction, content refinement, formatting, quality assurance checks, and error handling protocols.  
  - Input: Draft report from "Generator Agent" and output from "Evaluator Model".  
  - Output: Final polished research document with verification summary and logs.  
  - Failure Types: Complex prompt handling errors, incomplete evaluation.  

- **Sticky Note9 (Evaluate and Generate Final Report)**  
  - Purpose: Describes the evaluation chain’s comprehensive role in polishing and verifying the final output.  

---

## 3. Summary Table

| Node Name               | Node Type                                | Functional Role                                   | Input Node(s)           | Output Node(s)         | Sticky Note                                                     |
|-------------------------|----------------------------------------|-------------------------------------------------|------------------------|-----------------------|----------------------------------------------------------------|
| When chat message received | @n8n/n8n-nodes-langchain.chatTrigger | Receives user chat queries via webhook          | External webhook       | Search web            | # User query                                                   |
| Search web              | n8n-nodes-base.jinaAi                  | Performs Jina AI web search based on query      | When chat message received | Code                 | **Purpose:** Web search using Jina AI based on chat input. Requires Jina AI API key. |
| Code                    | n8n-nodes-base.code (Python)           | Extracts URLs from search results                | Search web             | Loop Over Items        | **Purpose:** Custom code to extract and format URLs for looping |
| Loop Over Items         | n8n-nodes-base.splitInBatches          | Iterates over each URL for individual processing| Code                   | Transform, Read URL content | **Purpose:** Processes each URL individually for content gathering and summarization |
| Read URL content        | n8n-nodes-base.jinaAi                  | Fetches full content of each URL                  | Loop Over Items         | Summarizer Agent       | **Purpose:** Fetches raw web page content for summarization     |
| Summarizer Agent        | @n8n/n8n-nodes-langchain.agent         | Summarizes webpage content using AI              | Read URL content        | Wait                   | **Purpose:** AI agent to summarize content from URLs           |
| Summarizer Model        | @n8n/n8n-nodes-langchain.lmChatGoogleGemini | Gemini 2.5 Flash model performing summarization | Summarizer Agent        | Summarizer Agent       | **Purpose:** Uses Gemini 2.5 Flash AI model for summarization  |
| Wait                    | n8n-nodes-base.wait                    | Delay to avoid API rate limits                    | Summarizer Agent        | Loop Over Items         | **Purpose:** Introduces pause to prevent hitting API rate limits |
| Transform               | n8n-nodes-base.code (Python)           | Aggregates all summaries into single object      | Loop Over Items         | Generator Agent        | **Purpose:** Consolidates summaries for report generation      |
| Generator Agent         | @n8n/n8n-nodes-langchain.agent         | Generates comprehensive research report          | Transform               | Evaluator Chain        | **Purpose:** AI agent creates detailed research report with citations |
| Generator Model         | @n8n/n8n-nodes-langchain.lmChatGoogleGemini | Gemini 2.5 Flash model generating report text   | Generator Agent         | Generator Agent        |                                                                |
| Evaluator Model         | @n8n/n8n-nodes-langchain.lmChatGoogleGemini | Gemini 2.5 Flash model evaluating report        | Generator Agent         | Evaluator Chain        |                                                                |
| Evaluator Chain         | @n8n/n8n-nodes-langchain.chainLlm      | Verifies, refines, and formats final report      | Generator Agent, Evaluator Model | —             | **Purpose:** Verifies citations, improves formatting, ensures quality |
| Sticky Note             | n8n-nodes-base.stickyNote               | Documentation notes                              | —                      | —                      | # User query                                                   |
| Sticky Note1            | n8n-nodes-base.stickyNote               | Documentation notes                              | —                      | —                      | # Search web documentation                                    |
| Sticky Note2            | n8n-nodes-base.stickyNote               | Documentation notes                              | —                      | —                      | # Code URL extraction explanation                             |
| Sticky Note3            | n8n-nodes-base.stickyNote               | Documentation notes                              | —                      | —                      | # Loop over URLs explanation                                  |
| Sticky Note4            | n8n-nodes-base.stickyNote               | Documentation notes                              | —                      | —                      | # Read URL content documentation                              |
| Sticky Note5            | n8n-nodes-base.stickyNote               | Documentation notes                              | —                      | —                      | # Summarize gathered content description                      |
| Sticky Note6            | n8n-nodes-base.stickyNote               | Documentation notes                              | —                      | —                      | # Avoid rate limits explanation                               |
| Sticky Note7            | n8n-nodes-base.stickyNote               | Documentation notes                              | —                      | —                      | # Transform summaries aggregator explanation                  |
| Sticky Note8            | n8n-nodes-base.stickyNote               | Documentation notes                              | —                      | —                      | # Generate research report explanation                        |
| Sticky Note9            | n8n-nodes-base.stickyNote               | Documentation notes                              | —                      | —                      | # Evaluate and generate final report explanation             |
| Sticky Note10           | n8n-nodes-base.stickyNote               | Documentation notes                              | —                      | —                      | Workflow overview and usage instructions                      |

---

## 4. Reproducing the Workflow from Scratch

1. **Create Webhook Trigger Node**  
   - Add node: `When chat message received` (LangChain Chat Trigger)  
   - Configure webhook with unique ID.  
   - No additional parameters needed.  

2. **Add Jina AI Search Node**  
   - Add node: `Search web` (Jina AI)  
   - Set operation to "search".  
   - Set `searchQuery` parameter as expression: `={{ $json.chatInput }}` to dynamically use chat input.  
   - Connect output of chat trigger to this node.  
   - Configure Jina AI API credentials with valid API key.  

3. **Add Python Code Node for URL Extraction**  
   - Add node: `Code` (Python)  
   - Paste code:  
     ```python
     extracted_urls = []
     for item in _input.all():
         original_json = item.json
         if 'url' in original_json:
             extracted_urls.append({'url': original_json['url']})
     return extracted_urls
     ```  
   - Connect output of "Search web" to this node.  

4. **Add Loop Over Items Node**  
   - Add node: `Loop Over Items` (SplitInBatches)  
   - Disable "reset" option.  
   - Connect output of "Code" node to this node.  

5. **Add Jina AI Content Fetch Node**  
   - Add node: `Read URL content` (Jina AI)  
   - Set parameter `url` as expression: `={{ $json.url }}`  
   - Connect output of "Loop Over Items" node to this node.  
   - Use the same Jina AI API credentials as before.  

6. **Add Summarizer Agent Node**  
   - Add node: `Summarizer Agent` (LangChain Agent)  
   - Configure prompt to summarize content using dynamic expressions for source url, title, description, content, and initial user query.  
   - Connect output of "Read URL content" to this node.  

7. **Add Summarizer Model Node**  
   - Add node: `Summarizer Model` (LangChain LM Chat Google Gemini)  
   - Set model name to `models/gemini-2.5-flash`.  
   - Connect AI language model input of "Summarizer Agent" to this node.  
   - Configure Google Palm API credentials.  

8. **Add Structured Output Parser Node**  
   - Add node: `Structured Output` (LangChain Output Parser Structured)  
   - Configure JSON schema example with keys `source_url` and `summarized_content`.  
   - Connect AI output parser input of "Summarizer Agent" to this node.  

9. **Add Wait Node**  
   - Add node: `Wait`  
   - Set wait time to 1 second to avoid API rate limits.  
   - Connect main output of "Summarizer Agent" to this node.  

10. **Connect Wait Node Back to Loop Over Items for Next URL**  
    - Connect output of "Wait" node to second output channel of "Loop Over Items" node (to continue loop).  

11. **Add Transform Node for Aggregating Summaries**  
    - Add node: `Transform` (Python)  
    - Paste code:  
      ```python
      all_outputs = []
      for item in _input.all():
          current_item_data = item.json
          output = current_item_data.get('output')
          if output:
              all_outputs.append(output)
      final_output_dict = {"output": all_outputs}
      return [{'json': final_output_dict}]
      ```  
    - Connect first output of "Loop Over Items" node to "Transform".  

12. **Add Generator Agent Node**  
    - Add node: `Generator Agent` (LangChain Agent)  
    - Paste the detailed prompt that instructs the AI to synthesize summaries into a comprehensive research report with citations and structure (as detailed in the original prompt).  
    - Connect output of "Transform" node to this node.  

13. **Add Generator Model Node**  
    - Add node: `Generator Model` (LangChain LM Chat Google Gemini)  
    - Set model name to `models/gemini-2.5-flash`.  
    - Connect AI language model input of "Generator Agent" to this node.  
    - Use Google Palm API credentials.  

14. **Add Evaluator Model Node**  
    - Add node: `Evaluator Model` (LangChain LM Chat Google Gemini)  
    - Set model name to `models/gemini-2.5-flash`.  
    - Connect AI language model input of "Evaluator Chain" node to this node.  
    - Use Google Palm API credentials.  

15. **Add Evaluator Chain Node**  
    - Add node: `Evaluator Chain` (LangChain Chain LLM)  
    - Paste the detailed prompt for citation verification, content quality assurance, markdown formatting, and professional report structure.  
    - Connect main output of "Generator Agent" to "Evaluator Chain".  
    - Connect AI language model output of "Evaluator Model" to "Evaluator Chain".  

16. **Final Output**  
    - The output of "Evaluator Chain" is the polished research report.  

---

## 5. General Notes & Resources

| Note Content                                                                                                               | Context or Link                                               |
|----------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------|
| Workflow intelligently processes chat messages by searching the web and generating AI-powered responses with Jina & Gemini. | Sticky Note10 documents overall workflow purpose and usage.    |
| Jina AI credentials are required for search and content fetching nodes.                                                    | Credential setup required for "Search web" and "Read URL content". |
| Google Palm API credentials must be configured for Gemini 2.5 Flash model nodes (Summarizer, Generator, Evaluator).         | Credential setup required for all LangChain Gemini model nodes. |
| Wait node configured with 1-second delay to avoid API rate limits, adjustable as needed depending on API constraints.       | Important for stable operation in content summarization loop.  |
| Custom Python code nodes are used for transforming and aggregating URLs and summaries, ensuring data structure consistency. | Node "Code" and "Transform" perform critical data shaping steps. |
| The Generator Agent prompt includes detailed instructions on research report structure, citation format, and quality standards. | Ensures output meets academic and professional criteria.       |
| The Evaluator Chain node performs citation verification and final formatting to produce a publication-ready document.       | Critical for ensuring output quality and correctness.          |

---

**Disclaimer:**  
The provided text and workflow content originate exclusively from an automated n8n workflow integrating Jina AI and Google Gemini APIs. This automation complies strictly with content policies and handles only legal, public data.

---