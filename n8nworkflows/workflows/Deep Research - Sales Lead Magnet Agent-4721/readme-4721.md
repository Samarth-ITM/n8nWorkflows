Deep Research - Sales Lead Magnet Agent

https://n8nworkflows.xyz/workflows/deep-research---sales-lead-magnet-agent-4721


# Deep Research - Sales Lead Magnet Agent

---
### 1. Workflow Overview

This workflow, titled **"Deep Research - Sales Lead Magnet Agent"**, is designed to automate the creation of deeply researched, AI-driven sales lead magnets focused on leveraging Trigify's platform capabilities or other user-specified topics. Its primary use case is to transform a user’s research query into a structured, actionable content asset optimized for LinkedIn sharing, authored in the style of Max Mitcham and branded by Trigify.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception and Query Refinement**  
  Receives user input via chat, refines the research topic into targeted search queries with contextual knowledge on Trigify if applicable.

- **1.2 Research and Content Planning**  
  Conducts in-depth research for each query, synthesizes insights, and generates a comprehensive table of contents for the lead magnet.

- **1.3 Chapter Writing (Team of Research Assistants)**  
  Splits the content plan into chapters and delegates writing tasks to AI agents, each producing detailed, actionable sections referencing Trigify knowledge.

- **1.4 Content Assembly and Editing**  
  Merges all chapter outputs, refines the full article text into a polished, authentic style matching Max Mitcham’s voice.

- **1.5 Document Generation and Sharing**  
  Creates and updates a Google Docs document with the final article, shares the file with appropriate permissions, and outputs the document URL.

Each block leverages LangChain agents, AI language models (Anthropic Claude), and specialized research tools (Perplexity, Trigify knowledge hub) integrated via n8n nodes to ensure research quality, relevance, and actionable content.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Query Refinement

- **Overview:**  
  This block captures the user's research topic and refines it into five focused, high-quality search queries, with special handling for Trigify-related topics using targeted knowledge tools.

- **Nodes Involved:**  
  - When chat message received  
  - Query Builder (LangChain Agent)  
  - Structured Output Parser

- **Node Details:**

  - **When chat message received**  
    - *Type:* Chat Trigger Node  
    - *Role:* Entry point; listens for incoming chat messages containing the user’s research query.  
    - *Configuration:* Default webhook-based trigger with no specific options.  
    - *Inputs:* Webhook HTTP POST from user chat interface.  
    - *Outputs:* Passes chat input text to Query Builder.  
    - *Failures:* Network issues, malformed requests.  

  - **Query Builder**  
    - *Type:* LangChain Agent Node  
    - *Role:* Refines user input into five targeted search queries.  
    - *Configuration:*  
      - Input expression: `{{$json.chatInput}}` (user query)  
      - System instructions define the agent as "Search Query Refiner Agent" with detailed logic to:  
        - Detect if topic involves Trigify  
        - Call Perplexity and Trigify knowledge hub tools for enhanced context on Trigify topics  
        - Generate exactly 5 focused queries in JSON format  
    - *Outputs:* Raw JSON string with topic and searchQueries array.  
    - *Failures:* Expression errors, API timeouts, tool invocation errors, malformed output.  
    - *Version:* 1.7  

  - **Structured Output Parser**  
    - *Type:* LangChain Structured Output Parser  
    - *Role:* Parses the JSON string output from Query Builder into structured JSON data for downstream nodes.  
    - *Configuration:* Uses a JSON schema example with "topic" and "searchQueries" fields.  
    - *Inputs:* Raw JSON string from Query Builder.  
    - *Outputs:* Parsed object with topic and searchQueries array.  
    - *Failures:* Parsing errors if output is malformed or incomplete.  
    - *Version:* 1.2  

---

#### 2.2 Research and Content Planning

- **Overview:**  
  Uses the refined search queries to perform deep research and generate a structured table of contents for the lead magnet article, tailored based on whether the topic is related to Trigify.

- **Nodes Involved:**  
  - Research Leader (LangChain Agent)  
  - Structured Output Parser1  
  - Project Planner (LangChain Agent)  
  - Perplexity tool (Tool Workflow)  
  - Trigify knowledge hub2 (HTTP Request Tool)  
  - Anthropic Chat Model1 (AI Language Model)  
  - OpenRouter Chat Model1 (AI Language Model)  

- **Node Details:**

  - **Research Leader**  
    - *Type:* LangChain Agent  
    - *Role:* Performs comprehensive research on the 5 queries, synthesizes insights, and drafts a detailed, research-backed table of contents for the article.  
    - *Configuration:*  
      - Text input combines the 5 queries from `searchQueries[]`.  
      - System prompt instructs use of Perplexity for non-Trigify topics and both Perplexity + Trigify knowledge hub2 for Trigify-related topics.  
      - Output is a structured TOC with insights and section breakdowns.  
    - *Inputs:* Parsed queries from Structured Output Parser.  
    - *Outputs:* JSON with detailed table of contents, including titles and prompts for each chapter.  
    - *Failures:* Model API errors, tool call failures, incomplete research data.  
    - *Version:* 1.7  

  - **Structured Output Parser1**  
    - *Type:* LangChain Structured Output Parser  
    - *Role:* Parses JSON output from Research Leader into usable structured data for Project Planner.  
    - *Configuration:* Uses example schema with `title`, `subtitle`, `introduction`, `conclusions`, `chapters` (array with title & prompt).  
    - *Failures:* Parsing errors if output format deviates.  
    - *Version:* 1.2  

  - **Project Planner**  
    - *Type:* LangChain Agent  
    - *Role:* Using the TOC, writes detailed article metadata (title, subtitle, introduction, conclusions) and chapter prompts for writing.  
    - *Configuration:*  
      - Input: TOC from Research Leader.  
      - System instructions emphasize actionable, deep research-oriented content with step-by-step guidance.  
      - Outputs JSON with article structure and chapter prompts.  
    - *Outputs:* JSON with article outline and chapter writing prompts.  
    - *Failures:* API call errors, output parsing problems.  
    - *Version:* 1.7  

  - **Perplexity tool**  
    - *Type:* Tool Workflow (Sub-workflow)  
    - *Role:* External research tool used by Research Leader and Project Planner for gathering general and non-Trigify research data.  
    - *Configuration:* Calls another workflow (`wPA7grIPtOHm3QjU`) designed for research queries.  
    - *Inputs:* Dynamic, passed from invoking nodes.  
    - *Outputs:* Research data.  
    - *Failures:* External API failures, timeouts.  
    - *Version:* 2  

  - **Trigify knowledge hub2**  
    - *Type:* HTTP Request Tool  
    - *Role:* Specialized knowledge source about Trigify, used for Trigify-related research queries.  
    - *Configuration:*  
      - POST request to Chatbase API with Claude 3.5 model.  
      - Authenticated via Bearer token.  
      - Content dynamically injected into JSON body for queries.  
    - *Inputs:* Queries from agents.  
    - *Outputs:* Knowledge responses.  
    - *Failures:* Auth errors, rate limits, malformed responses.  
    - *Version:* 1.1  

  - **Anthropic Chat Model1 & OpenRouter Chat Model1**  
    - *Type:* AI Language Model Nodes  
    - *Role:* Provide advanced language model capabilities to Research Leader and Project Planner for text generation and reasoning.  
    - *Configuration:* Claude 3.7-sonnet models with appropriate credentials.  
    - *Failures:* API quota limits, connection errors.  
    - *Version:* 1.2 & 1  

---

#### 2.3 Chapter Writing (Team of Research Assistants)

- **Overview:**  
  Splits the planned chapters and assigns each to a dedicated AI writing agent, producing detailed, actionable chapter texts using Trigify knowledge and research tools.

- **Nodes Involved:**  
  - Split Out  
  - Team of Research Assistants (LangChain Agent)  
  - Merge  

- **Node Details:**

  - **Split Out**  
    - *Type:* n8n Split Out Node  
    - *Role:* Splits the array of chapters from Project Planner into individual items for parallel processing.  
    - *Parameters:* Splits on `output.chapters`.  
    - *Inputs:* JSON array of chapters.  
    - *Outputs:* One item per chapter with title and prompt.  
    - *Failures:* Empty or malformed input array.  
    - *Version:* 1  

  - **Team of Research Assistants**  
    - *Type:* LangChain Agent  
    - *Role:* Writes the full text for each chapter based on the prompt, using Trigify knowledge hub and Perplexity research. Produces ~120 words per chapter in HTML format.  
    - *Configuration:*  
      - Input: Chapter title and prompt.  
      - System message instructs writing in actionable, step-by-step style, referencing Trigify knowledge hub.  
      - Citation guidelines included.  
      - Avoids repeating content from adjacent chapters for coherence.  
    - *Inputs:* One chapter at a time.  
    - *Outputs:* Plain text HTML content for each chapter.  
    - *Failures:* API errors, content generation failures.  
    - *Version:* 1.7  

  - **Merge**  
    - *Type:* n8n Merge Node  
    - *Role:* Combines all individual chapter texts back into a single collection for further processing.  
    - *Parameters:* Combine mode by position (preserves chapter order).  
    - *Inputs:* Multiple chapter outputs.  
    - *Outputs:* Combined array of chapter contents.  
    - *Failures:* Data mismatch if chapters are missing.  
    - *Version:* 3  

---

#### 2.4 Content Assembly and Editing

- **Overview:**  
  This block merges all chapters and edits the entire article into a polished, casual but formal style matching Max Mitcham’s voice, adding placeholders for images.

- **Nodes Involved:**  
  - Code (JavaScript)  
  - Editor (LangChain Agent)  

- **Node Details:**

  - **Code**  
    - *Type:* n8n Code Node (JavaScript)  
    - *Role:* Flattens and extracts the article title and all chapter texts into a single combined JSON structure for editing.  
    - *Configuration:*  
      - Collects all input JSON items, pushes title and output content into an array.  
      - Returns combinedData array containing title and chapter contents.  
    - *Inputs:* Array of chapter texts and metadata.  
    - *Outputs:* Single JSON with combined data.  
    - *Failures:* Runtime JS errors, empty inputs.  
    - *Version:* 2  

  - **Editor**  
    - *Type:* LangChain Agent  
    - *Role:* Edits and polishes the entire article text into a cohesive, authentic, and engaging narrative in Max Mitcham’s voice, suitable for LinkedIn sharing.  
    - *Configuration:*  
      - Input: Combined article data from Code node.  
      - System prompt: Expert editor guidelines emphasizing formal yet casual tone, consistency, grammar, SEO optimization, and placeholders for images.  
      - Output: Edited text JSON with final article content.  
    - *Failures:* Model errors, output parsing failures.  
    - *Version:* 1.7  

---

#### 2.5 Document Generation and Sharing

- **Overview:**  
  Creates a Google Docs document with the final article content, updates it, sets sharing permissions, and outputs a shareable URL.

- **Nodes Involved:**  
  - Google Docs  
  - Google Docs1 (Update)  
  - Google Drive (Sharing)  
  - Share URL (Set)  

- **Node Details:**

  - **Google Docs**  
    - *Type:* Google Docs Node  
    - *Role:* Creates a new Google Doc titled with the article title from Project Planner output.  
    - *Configuration:*  
      - Title: Dynamic from `Project Planner.output.title`  
      - Folder ID: Fixed Google Drive folder ID for document storage.  
    - *Inputs:* Title from workflow JSON.  
    - *Outputs:* Document metadata including document ID.  
    - *Failures:* Auth errors, quota limits.  
    - *Version:* 2  

  - **Google Docs1**  
    - *Type:* Google Docs Node (Update)  
    - *Role:* Inserts the fully edited article text into the newly created Google Doc.  
    - *Configuration:*  
      - Operation: Update  
      - Document URL: From previous node's document ID  
      - Actions: Insert text content from Editor output.  
    - *Failures:* Auth errors, malformed content.  
    - *Version:* 2  

  - **Google Drive**  
    - *Type:* Google Drive Node  
    - *Role:* Sets sharing permissions on the Google Doc to "Anyone with the link can view."  
    - *Configuration:*  
      - File ID: Document ID from Google Docs1  
      - Permissions: Role = reader, Type = anyone  
    - *Failures:* Permission errors, API limits.  
    - *Version:* 3  

  - **Share URL**  
    - *Type:* Set Node  
    - *Role:* Constructs and outputs a shareable Google Docs URL using the document ID for downstream consumption or notification.  
    - *Configuration:*  
      - Sets field with URL format: `https://docs.google.com/document/d/{DocumentId}`  
      - Passes through some webhook body fields (First Name, Last Name, etc.) if present.  
    - *Failures:* None expected.  
    - *Version:* 3.4  

---

### 3. Summary Table

| Node Name                | Node Type                          | Functional Role                                     | Input Node(s)             | Output Node(s)          | Sticky Note                                                                                             |
|--------------------------|----------------------------------|----------------------------------------------------|---------------------------|-------------------------|--------------------------------------------------------------------------------------------------------|
| When chat message received| @n8n/n8n-nodes-langchain.chatTrigger | Entry point: receives user research query           | -                         | Query Builder           |                                                                                                        |
| Query Builder            | @n8n/n8n-nodes-langchain.agent   | Refines user input into 5 targeted search queries  | When chat message received | Structured Output Parser | Contains detailed system instructions for Trigify topic detection and query refinement.                 |
| Structured Output Parser  | @n8n/n8n-nodes-langchain.outputParserStructured | Parses JSON output from Query Builder                | Query Builder             | Research Leader         |                                                                                                        |
| Research Leader          | @n8n/n8n-nodes-langchain.agent   | Performs deep research on queries, outputs TOC     | Structured Output Parser  | Project Planner         | Uses Perplexity and Trigify knowledge hub2 for research.                                               |
| Structured Output Parser1 | @n8n/n8n-nodes-langchain.outputParserStructured | Parses Research Leader output JSON                    | Research Leader           | Project Planner         |                                                                                                        |
| Project Planner          | @n8n/n8n-nodes-langchain.agent   | Creates article metadata and chapter prompts        | Research Leader           | Split Out               | Outputs article structure with chapters and prompts.                                                   |
| Perplexity tool          | @n8n/n8n-nodes-langchain.toolWorkflow | External research tool called by agents             | Various                   | Various                 | Sub-workflow for online research.                                                                     |
| Trigify knowledge hub2   | @n8n/n8n-nodes-langchain.toolHttpRequest | Specialized Trigify knowledge source                 | Various                   | Various                 | Authenticated HTTP requests to Chatbase API using Claude model.                                       |
| Anthropic Chat Model1    | @n8n/n8n-nodes-langchain.lmChatAnthropic | AI language model for research and planning          | Various                   | Various                 |                                                                                                        |
| OpenRouter Chat Model1   | @n8n/n8n-nodes-langchain.lmChatOpenRouter | Alternative AI language model                         | Various                   | Various                 |                                                                                                        |
| Split Out                | n8n-nodes-base.splitOut          | Splits chapters array for parallel processing       | Project Planner           | Team of Research Assistants, Merge |                                                                                                        |
| Team of Research Assistants | @n8n/n8n-nodes-langchain.agent | Writes chapter texts based on prompts                | Split Out                 | Merge                   | Uses Trigify knowledge hub and Perplexity for research; writes ~120 words per chapter in HTML.         |
| Merge                    | n8n-nodes-base.merge             | Combines all chapter texts into single collection   | Team of Research Assistants | Code                    |                                                                                                        |
| Code                     | n8n-nodes-base.code              | Aggregates title and chapter contents                | Merge                     | Editor                  | Combines data array for final editing.                                                                |
| Editor                   | @n8n/n8n-nodes-langchain.agent   | Edits and polishes full article into authentic style| Code                      | Google Docs             | Adds placeholders for images; formal but casual style per Max Mitcham.                                |
| Google Docs              | n8n-nodes-base.googleDocs        | Creates new Google Doc for article                    | Editor                    | Google Docs1            |                                                                                                        |
| Google Docs1             | n8n-nodes-base.googleDocs        | Updates Google Doc with full article content         | Google Docs               | Google Drive            |                                                                                                        |
| Google Drive             | n8n-nodes-base.googleDrive       | Sets sharing permissions on Google Doc               | Google Docs1              | Share URL               |                                                                                                        |
| Share URL                | n8n-nodes-base.set               | Outputs shareable Google Docs URL with metadata      | Google Drive              | -                       | Passes some webhook body fields for potential notification or logging.                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Chat Trigger Node**  
   - Type: `@n8n/n8n-nodes-langchain.chatTrigger`  
   - Purpose: Receive user’s research topic as chat input.  
   - Configure webhook ID and default options.  

2. **Add Query Builder Node**  
   - Type: `@n8n/n8n-nodes-langchain.agent`  
   - Connect input from Chat Trigger.  
   - Set `text` parameter to `={{ $json.chatInput }}`.  
   - Provide system instructions to refine topic into 5 search queries focusing on Trigify context when applicable.  
   - Enable use of Perplexity and Trigify knowledge hub2 tools internally.  

3. **Add Structured Output Parser Node**  
   - Type: `@n8n/n8n-nodes-langchain.outputParserStructured`  
   - Connect input from Query Builder.  
   - Define JSON schema example with fields: `topic` (string), `searchQueries` (array of 5 strings).  

4. **Add Research Leader Node**  
   - Type: `@n8n/n8n-nodes-langchain.agent`  
   - Connect input from Structured Output Parser.  
   - Configure prompt to perform deep research on all 5 queries, with instructions to:  
     - Use Perplexity tool for general research  
     - Use Trigify knowledge hub2 for Trigify topics  
     - Produce structured table of contents with key insights.  
   - Attach Anthropic or OpenRouter model credentials.  

5. **Add Structured Output Parser1 Node**  
   - Type: `@n8n/n8n-nodes-langchain.outputParserStructured`  
   - Connect input from Research Leader.  
   - Define JSON schema with article `title`, `subtitle`, `introduction`, `conclusions`, and `chapters` array (each with `title` and `prompt`).  

6. **Add Project Planner Node**  
   - Type: `@n8n/n8n-nodes-langchain.agent`  
   - Connect input from Structured Output Parser1.  
   - Configure prompt to create detailed article structure and chapter prompts, instructing for actionable, step-by-step content.  
   - Use Anthropic or OpenRouter model credentials.  

7. **Add Split Out Node**  
   - Type: `n8n-nodes-base.splitOut`  
   - Connect input from Project Planner.  
   - Set `fieldToSplitOut` to `output.chapters` to process chapters individually.  

8. **Add Team of Research Assistants Node**  
   - Type: `@n8n/n8n-nodes-langchain.agent`  
   - Connect input from Split Out.  
   - Configure prompt to write ~120 words per chapter, referencing Trigify knowledge hub, producing actionable text.  
   - Use appropriate AI credentials.  

9. **Add Merge Node**  
   - Type: `n8n-nodes-base.merge`  
   - Connect multiple outputs from Team of Research Assistants.  
   - Set mode to `combine` by position to maintain chapter order.  

10. **Add Code Node**  
    - Type: `n8n-nodes-base.code`  
    - Connect input from Merge.  
    - JS code to aggregate title and chapter texts into a single combined JSON object.  

11. **Add Editor Node**  
    - Type: `@n8n/n8n-nodes-langchain.agent`  
    - Connect input from Code.  
    - Configure prompt to polish full article into Max Mitcham’s formal yet casual style, add image placeholders.  
    - Use Anthropic or OpenRouter model credentials.  

12. **Add Google Docs Node (Create)**  
    - Type: `n8n-nodes-base.googleDocs`  
    - Connect input from Editor.  
    - Set title dynamically from Project Planner article title.  
    - Specify target folder ID for document storage.  
    - Configure Google Docs OAuth2 credentials.  

13. **Add Google Docs1 Node (Update)**  
    - Type: `n8n-nodes-base.googleDocs`  
    - Connect input from Google Docs.  
    - Operation: Update document content with edited article text from Editor.  
    - Use document ID from previous node.  

14. **Add Google Drive Node (Share)**  
    - Type: `n8n-nodes-base.googleDrive`  
    - Connect input from Google Docs1.  
    - Operation: Share document with "anyone with the link" as reader.  
    - Use Google Drive OAuth2 credentials.  

15. **Add Set Node (Share URL)**  
    - Type: `n8n-nodes-base.set`  
    - Connect input from Google Drive.  
    - Create field with shareable Google Docs URL based on document ID.  
    - Optionally pass through webhook body fields for context.  

16. **Credential Setup:**  
    - Configure OpenAI or Anthropic API credentials for language models.  
    - Configure Google OAuth2 credentials for Docs and Drive.  
    - Configure any API keys for Trigify knowledge hub and Perplexity tools.  

17. **Sub-Workflow Setup:**  
    - Ensure Perplexity tool sub-workflow is imported and connected.  
    - Verify Trigify knowledge hub HTTP request node is using valid Bearer token and endpoint.  

This stepwise configuration allows complete reproduction of the workflow, enabling ingestion of a user topic, refinement into queries, deep research, content planning, chapter writing, editing, and final document sharing.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                           | Context or Link                                                                                         |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| The workflow is designed for creating LinkedIn-optimized lead magnets branded by Trigify and authored in Max Mitcham’s voice.                                                                                                        | Branding context from system prompts.                                                                 |
| Perplexity and Trigify knowledge hub2 are key knowledge tools integrated for research, ensuring accurate and relevant content.                                                                                                       | Workflow nodes "Perplexity tool" and "Trigify knowledge hub2."                                        |
| The AI models used are Claude 3.5 and 3.7 (Sonnet versions), Anthropic and OpenRouter, requiring valid API keys and quotas.                                                                                                         | Anthropic Chat Model nodes and OpenRouter Chat Model1 node.                                           |
| Google Docs and Drive are integrated for document creation, update, and sharing using OAuth2 credentials.                                                                                                                             | Google Docs and Google Drive nodes.                                                                   |
| The workflow includes advanced structured output parsing to ensure all AI-generated JSON outputs are properly interpreted, minimizing errors.                                                                                      | Structured Output Parser nodes.                                                                        |
| Content style instructions emphasize actionable, step-by-step guidance with practical examples and troubleshooting tips for sales teams using Trigify.                                                                              | System messages for Project Planner and Team of Research Assistants nodes.                             |
| Sample references and citations are embedded as HTML links in chapter texts to maintain source attribution and credibility.                                                                                                          | Citation guidelines in Team of Research Assistants node.                                              |
| Image placeholders are added during editing with `{Add image here of X}` syntax for later manual enrichment.                                                                                                                         | Editor node instructions.                                                                              |
| The workflow is inactive (`active: false`) and requires activation and credential setup before use.                                                                                                                                  | Workflow metadata.                                                                                    |
| The workflow is structured for linear execution with one main entry point (chat trigger) and no parallel branches except for chapter writing, which is parallelized for efficiency.                                                   | Workflow connections overview.                                                                        |

---

**Disclaimer:** The text above is exclusively derived from an automated n8n workflow designed for legal and public data processing. It contains no illegal, offensive, or protected content. All data handled complies with applicable content policies.