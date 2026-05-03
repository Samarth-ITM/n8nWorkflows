Full Blog Content Automation with GPT-4, Claude & Ghost CMS Publisher

https://n8nworkflows.xyz/workflows/full-blog-content-automation-with-gpt-4--claude---ghost-cms-publisher-7920


# Full Blog Content Automation with GPT-4, Claude & Ghost CMS Publisher

### 1. Workflow Overview

This workflow automates the entire blogging process from a user-provided topic to a fully researched, SEO-optimized, edited, and published blog post on a Ghost CMS website. It is designed for content teams or solo creators who want to leverage AI agents to streamline blog content creation with minimal manual intervention while maintaining high quality and SEO standards.

The workflow is logically divided into the following functional blocks:

- **1.1 Input Reception and Orchestration**  
  Handles the user input (blog topic and requirements), orchestrates the workflow phases, coordinates between specialized AI agents, and manages approval gates.

- **1.2 Research and Outline Generation**  
  Conducts comprehensive multi-source research using Brave Search, Brave News, and existing blog content; produces a well-structured and SEO-optimized Table of Contents (TOC) and research insights.

- **1.3 Content Generation**  
  Transforms the approved TOC and research insights into a full blog draft with citations, internal links, and consistent style.

- **1.4 SEO Optimization**  
  Performs keyword research, on-page SEO enhancements, technical SEO analysis, and content improvement recommendations to maximize search visibility.

- **1.5 Content Editing and Quality Assurance**  
  Polishes the SEO-optimized draft for readability, citation accuracy, flow, and web formatting, ensuring professional HTML output ready for publication.

- **1.6 HTML Generation and Ghost CMS Publishing**  
  Generates complete SEO-friendly HTML documents with meta tags and schema markup and publishes the final content as a draft post in Ghost CMS via the Ghost Publisher tool.

- **1.7 Supporting Tools and Memory Management**  
  Supports the workflow via memory buffers for context retention and multiple language model nodes for redundancy and failover.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Orchestration Block

- **Overview:**  
  This block receives the initial user chat message (blog topic and instructions) and coordinates the entire blogging workflow. It validates inputs, delegates tasks to specialized agents, manages user approval points, and ensures the sequential flow of the process.

- **Nodes Involved:**  
  - When chat message received  
  - Blog Content Orchestrator Agent  
  - Simple Memory1

- **Node Details:**

  - **When chat message received**  
    - Type: LangChain Chat Trigger (Webhook)  
    - Role: Entry point for user blog topic input via chat interface; supports streaming response mode.  
    - Config: Public webhook, streaming enabled for real-time interaction.  
    - Input: Incoming chat messages from users.  
    - Output: Passes chat input to Blog Content Orchestrator Agent.  
    - Potential Failures: Webhook connectivity issues, malformed inputs.

  - **Blog Content Orchestrator Agent**  
    - Type: LangChain Agent  
    - Role: Central coordinator managing blog creation phases (research, TOC approval, content generation, SEO, editing, publishing).  
    - Config: Uses OpenAI GPT-4 and Anthropic Claude models as language models; contains detailed system prompts describing phases, validation steps, and error handling.  
    - Input: User chat message and parameters extracted.  
    - Output: Delegates to Research Agent and subsequent agents; awaits user approvals.  
    - Expressions/Variables: Uses system variables for blog style, chapter count, word count, current date, and target audience.  
    - Failures: Missing or invalid user input parameters, agent unavailability, expression evaluation errors.

  - **Simple Memory1**  
    - Type: LangChain Memory Buffer Window  
    - Role: Maintains conversational context memory for the orchestrator agent (context window length 20).  
    - Input/Output: Stores recent dialogue history to support coherent multi-turn interactions.  
    - Potential Issues: Memory overflow if context too large; synchronization delays.

---

#### 2.2 Research and Outline Generation Block

- **Overview:**  
  Conducts in-depth research across Brave Search, Brave News, and existing blog content. Synthesizes findings into a comprehensive, SEO-optimized Table of Contents (TOC) and research insights, enforcing mandatory delays between search requests to comply with API rate limits.

- **Nodes Involved:**  
  - Research Agent  
  - Brave Search (multiple instances)  
  - Brave News  
  - Blog Content tool  
  - OpenAI Chat Model4  
  - Anthropic Chat Model1  
  - Simple Memory  
  - Sticky Note1 (Documentation)

- **Node Details:**

  - **Research Agent**  
    - Type: LangChain Agent Tool  
    - Role: Manages multi-phase research and TOC creation, integrating multiple data sources with enforced 2-second delays between Brave Search queries.  
    - Config: Complex system prompt guiding research methodology, phase sequencing, and quality assurance checklist.  
    - Input: User parameters including blog topic, style, section count, target word count, date, additional context, and audience.  
    - Output: Returns structured TOC markdown and research insights for approval.  
    - Failures: Brave Search API rate limiting or timeouts, incomplete input data, logic errors in prompt.

  - **Brave Search Nodes** (Brave Search, Brave Search2, Brave Search3, Brave Search4)  
    - Type: Brave Search API Integration  
    - Role: Executes search queries with dynamic inputs from AI overrides for various research phases (core topic, trending subtopics, section research).  
    - Config: Queries are dynamically generated from AI outputs using expressions.  
    - Input: Search query strings, credentials for Brave Search API.  
    - Output: Search results passed back to Research Agent or other agents.  
    - Failures: API rate limiting, connectivity issues, malformed queries.

  - **Brave News**  
    - Type: Brave News Search API  
    - Role: Retrieves recent news relevant to the blog topic to capture current developments.  
    - Config: Search query dynamically set from AI-generated input.  
    - Failure Modes: Same as Brave Search.

  - **Blog Content tool**  
    - Type: Ghost CMS Content API Tool  
    - Role: Fetches all existing blog content in plaintext format for internal content analysis to avoid duplication and identify gaps.  
    - Input: Ghost Content API credentials.  
    - Output: Content data for Research Agent.  
    - Failures: API unavailability, credential errors.

  - **OpenAI Chat Model4 & Anthropic Chat Model1**  
    - Type: LangChain Language Models (OpenAI GPT-4 and Anthropic Claude)  
    - Role: Used by Research Agent for expert analysis and synthesis of research data into insights and TOC.  
    - Credentials: OpenAI and Anthropic API keys.  
    - Failures: API call failures, rate limits, invalid prompt formatting.

  - **Simple Memory**  
    - Type: LangChain Memory Buffer Window  
    - Role: Maintains conversation context for Research Agent.  
    - Config: Default settings.  
    - Issues: Same as other memory nodes.

---

#### 2.3 Content Generation Block

- **Overview:**  
  Converts the approved TOC and research summaries into a fully fleshed out blog draft. Conducts section-by-section research with enforced delays, integrates citations and internal links, and maintains consistent style and tone.

- **Nodes Involved:**  
  - Blog Content Generation Agent  
  - Brave Search3  
  - Blog Content2  
  - OpenAI Chat Model5  
  - Anthropic Chat Model2  
  - Simple Memory4  
  - Sticky Note2 (Documentation)

- **Node Details:**

  - **Blog Content Generation Agent**  
    - Type: LangChain Agent Tool  
    - Role: Master content writer creating research-backed, well-cited blog sections based on the approved TOC and insights.  
    - Input: Approved TOC, research insights, SEO keywords, blog style, target word count, section count, current date, source research, blog topic, target audience.  
    - Output: Complete blog draft in HTML format with embedded citations and internal links.  
    - Delay Handling: Enforces minimum 2-second delay between Brave Search requests per section.  
    - Failures: API or agent errors, missing input data, rate limits.

  - **Brave Search3**  
    - Role: Provides section-specific research data to content generation agent with dynamic queries.  
    - Same details as other Brave Search nodes.

  - **Blog Content2 (Ghost Content tool)**  
    - Role: Provides internal blog content for citation and internal linking research.  
    - Output used by content generator.

  - **OpenAI Chat Model5 & Anthropic Chat Model2**  
    - Role: Language models used for writing and refining blog content.  
    - Failures: Same as above.

  - **Simple Memory4**  
    - Role: Maintains memory context for content generation conversations.

---

#### 2.4 SEO Optimization Block

- **Overview:**  
  Performs comprehensive SEO optimization including keyword research, on-page optimization, meta tag generation, and content enhancement recommendations. Ensures content maximizes search visibility and ranking potential.

- **Nodes Involved:**  
  - SEO Optimizer Agent  
  - Brave Search  
  - OpenAI Chat Model  
  - Anthropic Chat Model5  
  - Simple Memory  
  - Sticky Note3 (Documentation)

- **Node Details:**

  - **SEO Optimizer Agent**  
    - Type: LangChain Agent Tool  
    - Role: SEO specialist agent that performs keyword research, analyzes competitor content, optimizes meta tags, headers, URLs, schema markup, and internal linking strategy.  
    - Input: Complete blog draft, TOC, blog topic, audience, style, word count, research date, and previous SEO keywords.  
    - Output: SEO-optimized content and detailed SEO analysis report.  
    - Delay Handling: Enforces 2-second delays between Brave Search requests for keyword research.  
    - Failures: Brave Search unavailability, incomplete payloads, API errors.

  - **Brave Search**  
    - Role: Supports SEO keyword research and competitor analysis.

  - **OpenAI Chat Model & Anthropic Chat Model5**  
    - Role: Language models supporting SEO analysis and content enhancement.

  - **Simple Memory**  
    - Role: Maintains SEO conversation context.

---

#### 2.5 Content Editing and Quality Assurance Block

- **Overview:**  
  Polishes the SEO-optimized blog draft by improving readability, verifying citations, enhancing flow, and formatting content with professional HTML. Ensures all links are functional and content is publication-ready.

- **Nodes Involved:**  
  - Blog Content Editor Agent  
  - Brave Search4  
  - Blog Content1  
  - OpenAI Chat Model3  
  - Anthropic Chat Model3  
  - Sticky Note4 (Documentation)

- **Node Details:**

  - **Blog Content Editor Agent**  
    - Type: LangChain Agent Tool  
    - Role: Senior editor agent refining blog draft for clarity, citation accuracy, flow, and HTML formatting.  
    - Input: SEO-optimized content, SEO analysis report, TOC, style, word count, section count, current date, keyword strategy, optimization scores, recommendations, editing instructions.  
    - Output: Final polished HTML content ready for publishing.  
    - Delay Handling: Uses Brave Search with enforced delays for additional source verification if needed.  
    - Failures: Missing required fields, broken links, citation gaps.

  - **Brave Search4**  
    - Role: Supports additional citation verification and research during editing.

  - **Blog Content1 (Ghost Content tool)**  
    - Role: Provides internal blog content for citation and link verification during editing.

  - **OpenAI Chat Model3 & Anthropic Chat Model3**  
    - Role: Language models assisting in editing and content refinement.

---

#### 2.6 HTML Generation and Ghost CMS Publishing Block

- **Overview:**  
  Generates full SEO-optimized HTML documents with proper meta tags, schema markup, and social media tags. Publishes the final blog post draft to Ghost CMS via Ghost Publisher tool, ensuring all publication metadata is correctly set.

- **Nodes Involved:**  
  - Blog HTML Publisher Agent  
  - Ghost publisher tool  
  - Simple Memory2  
  - OpenAI Chat Model6  
  - Anthropic Chat Model4  
  - Sticky Note5 (Documentation)

- **Node Details:**

  - **Blog HTML Publisher Agent**  
    - Type: LangChain Agent Tool  
    - Role: Web publication specialist generating final HTML and executing the Ghost Publisher tool to create a draft post in Ghost CMS.  
    - Input: Final edited content, TOC, style, word count, section count, current date, editing summary, QA metrics, SEO analysis, keyword strategy.  
    - Output: Complete HTML document and confirmation of Ghost post creation.  
    - Critical Step: Must execute Ghost Publisher tool node to publish content; this is mandatory.  
    - Failures: Ghost API errors, invalid HTML, missing required fields.

  - **Ghost publisher tool**  
    - Type: Ghost CMS Admin API Node  
    - Role: Performs actual creation of a blog post in Ghost CMS with draft status, including meta title, description, slug, and content.  
    - Input: Parameters dynamically injected via AI expressions.  
    - Output: Ghost CMS post creation confirmation.  
    - Failures: API authentication failure, network issues, invalid content format.

  - **Simple Memory2**  
    - Role: Maintains memory context for HTML publishing conversations.

  - **OpenAI Chat Model6 & Anthropic Chat Model4**  
    - Role: Language models supporting HTML generation and publication metadata creation.

---

#### 2.7 Supporting Tools and Memory Management Block

- **Overview:**  
  Includes multiple Simple Memory nodes for conversational context retention and multiple OpenAI and Anthropic chat model nodes providing redundancy and failover options.

- **Nodes Involved:**  
  - Simple Memory (multiple instances)  
  - OpenAI Chat Model (multiple versions)  
  - Anthropic Chat Model (multiple versions)

- **Node Details:**  
  - These nodes maintain context windows of recent interactions to ensure coherent multi-turn conversations between user and agents and provide multiple language model options to improve reliability and performance.

---

### 3. Summary Table

| Node Name                   | Node Type                           | Functional Role                          | Input Node(s)                          | Output Node(s)                         | Sticky Note                                                                                     |
|-----------------------------|-----------------------------------|----------------------------------------|--------------------------------------|---------------------------------------|------------------------------------------------------------------------------------------------|
| When chat message received  | LangChain Chat Trigger            | Entry point for user input             | -                                    | Blog Content Orchestrator Agent       | 🚀 Publisher Agent: Handles publication technical details and Ghost CMS publishing              |
| Blog Content Orchestrator Agent | LangChain Agent                  | Workflow coordinator and phase manager | When chat message received            | Research Agent, SEO Optimizer Agent, Blog Content Generation Agent, Blog Content Editor Agent, Blog HTML Publisher Agent | 🤖 Blog Orchestrator Agent: The Command Center orchestrating entire blog workflow             |
| Simple Memory1              | LangChain Memory Buffer Window    | Context memory for orchestrator        | Blog Content Orchestrator Agent      | Blog Content Orchestrator Agent       |                                                                                                |
| Research Agent              | LangChain Agent Tool              | Multi-source research and TOC creation | Blog Content Orchestrator Agent      | Blog Content Orchestrator Agent       | 🔍 Research Agent: Conducts research and creates outline                                        |
| Brave Search                | Brave Search API Integration      | Research queries for SEO Optimizer     | SEO Optimizer Agent                  | SEO Optimizer Agent                    | 🎯 SEO Optimizer Agent: SEO keyword research and optimization                                  |
| Brave Search2               | Brave Search API Integration      | Research queries for Research Agent    | Research Agent                      | Research Agent                        | 🔍 Research Agent: Conducts research and creates outline                                        |
| Brave Search3               | Brave Search API Integration      | Section research for Content Generator | Blog Content Generation Agent        | Blog Content Generation Agent         | ✍️ Content Generator Agent: Writes well-cited blog content                                    |
| Brave Search4               | Brave Search API Integration      | Research for Content Editor             | Blog Content Editor Agent             | Blog Content Editor Agent              | ✂️ Content Editor Agent: Polishes content, verifies citations                                |
| Brave News                  | Brave News Search API             | Recent news retrieval for Research Agent | Research Agent                      | Research Agent                        | 🔍 Research Agent: Conducts research and creates outline                                        |
| Blog Content tool           | Ghost CMS Content API Tool        | Retrieves existing blog content         | Research Agent, Blog Content Generation Agent, Blog Content Editor Agent | Respective agents                     |                                                                                                |
| OpenAI Chat Model           | LangChain OpenAI Chat Model       | Language model for SEO Optimizer Agent | SEO Optimizer Agent                  | SEO Optimizer Agent                    | 🎯 SEO Optimizer Agent: SEO keyword research and optimization                                  |
| OpenAI Chat Model1          | LangChain OpenAI Chat Model       | Language model for Orchestrator         | Blog Content Orchestrator Agent      | Blog Content Orchestrator Agent       | 🤖 Blog Orchestrator Agent: The Command Center orchestrating entire blog workflow             |
| OpenAI Chat Model3          | LangChain OpenAI Chat Model       | Language model for Content Editor       | Blog Content Editor Agent             | Blog Content Editor Agent              | ✂️ Content Editor Agent: Polishes content, verifies citations                                |
| OpenAI Chat Model4          | LangChain OpenAI Chat Model       | Language model for Research Agent       | Research Agent                      | Research Agent                        | 🔍 Research Agent: Conducts research and creates outline                                        |
| OpenAI Chat Model5          | LangChain OpenAI Chat Model       | Language model for Content Generator    | Blog Content Generation Agent        | Blog Content Generation Agent         | ✍️ Content Generator Agent: Writes well-cited blog content                                    |
| OpenAI Chat Model6          | LangChain OpenAI Chat Model       | Language model for HTML Publisher        | Blog HTML Publisher Agent            | Blog HTML Publisher Agent             | 🚀 Publisher Agent: Handles publication technical details and Ghost CMS publishing              |
| Anthropic Chat Model        | LangChain Anthropic Chat Model    | Alternate language model for Orchestrator | Blog Content Orchestrator Agent      | Blog Content Orchestrator Agent       | 🤖 Blog Orchestrator Agent: The Command Center orchestrating entire blog workflow             |
| Anthropic Chat Model1       | LangChain Anthropic Chat Model    | Alternate language model for Research Agent | Research Agent                      | Research Agent                        | 🔍 Research Agent: Conducts research and creates outline                                        |
| Anthropic Chat Model2       | LangChain Anthropic Chat Model    | Alternate language model for Content Generator | Blog Content Generation Agent        | Blog Content Generation Agent         | ✍️ Content Generator Agent: Writes well-cited blog content                                    |
| Anthropic Chat Model3       | LangChain Anthropic Chat Model    | Alternate language model for Content Editor | Blog Content Editor Agent             | Blog Content Editor Agent              | ✂️ Content Editor Agent: Polishes content, verifies citations                                |
| Anthropic Chat Model4       | LangChain Anthropic Chat Model    | Alternate language model for HTML Publisher | Blog HTML Publisher Agent            | Blog HTML Publisher Agent             | 🚀 Publisher Agent: Handles publication technical details and Ghost CMS publishing              |
| Anthropic Chat Model5       | LangChain Anthropic Chat Model    | Alternate language model for SEO Optimizer | SEO Optimizer Agent                  | SEO Optimizer Agent                    | 🎯 SEO Optimizer Agent: SEO keyword research and optimization                                  |
| Simple Memory              | LangChain Memory Buffer Window    | Context memory for SEO Optimizer Agent  | SEO Optimizer Agent                  | SEO Optimizer Agent                    |                                                                                                |
| Simple Memory2             | LangChain Memory Buffer Window    | Context memory for HTML Publisher Agent | Blog HTML Publisher Agent            | Blog HTML Publisher Agent             |                                                                                                |
| Simple Memory4             | LangChain Memory Buffer Window    | Context memory for Content Generator Agent | Blog Content Generation Agent        | Blog Content Generation Agent         |                                                                                                |
| Blog Content Generation Agent | LangChain Agent Tool              | Generates full blog content from TOC and research | Blog Content Orchestrator Agent      | Blog Content Orchestrator Agent       | ✍️ Content Generator Agent: Writes well-cited blog content                                    |
| Blog Content Editor Agent  | LangChain Agent Tool              | Edits and polishes blog draft           | Blog Content Orchestrator Agent       | Blog Content Orchestrator Agent       | ✂️ Content Editor Agent: Polishes content, verifies citations                                |
| SEO Optimizer Agent        | LangChain Agent Tool              | Performs SEO keyword research and optimization | Blog Content Orchestrator Agent       | Blog Content Orchestrator Agent       | 🎯 SEO Optimizer Agent: SEO keyword research and optimization                                  |
| Blog HTML Publisher Agent  | LangChain Agent Tool              | Generates HTML and publishes to Ghost CMS | Blog Content Orchestrator Agent       | Blog Content Orchestrator Agent       | 🚀 Publisher Agent: Handles publication technical details and Ghost CMS publishing              |
| Ghost publisher tool       | Ghost Admin API Node               | Publishes the blog post draft to Ghost CMS | Blog HTML Publisher Agent            | Blog HTML Publisher Agent             | 🚀 Publisher Agent: Handles publication technical details and Ghost CMS publishing              |
| Sticky Note                | Sticky Note                      | Documentation for Blog Content Orchestrator | -                                    | -                                     | 🤖 Blog Orchestrator Agent: The Command Center orchestrating entire blog workflow             |
| Sticky Note1               | Sticky Note                      | Documentation for Research Agent        | -                                    | -                                     | 🔍 Research Agent: Conducts research and creates outline                                      |
| Sticky Note2               | Sticky Note                      | Documentation for Content Generator Agent | -                                    | -                                     | ✍️ Content Generator Agent: Writes well-cited blog content                                    |
| Sticky Note3               | Sticky Note                      | Documentation for SEO Optimizer Agent   | -                                    | -                                     | 🎯 SEO Optimizer Agent: SEO keyword research and optimization                                |
| Sticky Note4               | Sticky Note                      | Documentation for Content Editor Agent  | -                                    | -                                     | ✂️ Content Editor Agent: Polishes content, verifies citations                              |
| Sticky Note5               | Sticky Note                      | Documentation for Blog HTML Publisher Agent | -                                    | -                                     | 🚀 Publisher Agent: Handles publication technical details and Ghost CMS publishing            |
| Sticky Note6               | Sticky Note                      | Overall workflow documentation and setup | -                                    | -                                     | 🌟 Complete Workflow Overview with setup requirements                                       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Trigger Node:**  
   - Type: LangChain Chat Trigger  
   - Configure as public webhook with streaming enabled.  
   - Name: "When chat message received"

2. **Add Blog Content Orchestrator Agent Node:**  
   - Type: LangChain Agent  
   - Connect input from "When chat message received" node.  
   - Configure with system prompt defining roles, phases, variables (blog style, chapter count, etc.), user interaction flow, validation, error handling, and approvals.  
   - Attach OpenAI GPT-4 and Anthropic Claude API credentials.  
   - Enable streaming output.  
   - Connect Simple Memory1 node for context retention.

3. **Add Simple Memory1 Node:**  
   - Type: LangChain Memory Buffer Window  
   - Configure with context window length 20.  
   - Connect to Blog Content Orchestrator Agent for memory management.

4. **Add Research Agent Node:**  
   - Type: LangChain Agent Tool  
   - Connect input from Blog Content Orchestrator Agent.  
   - Configure detailed system prompt for multi-phase research, TOC creation, Brave Search and Brave News integration, delay enforcement (min 2 seconds), error handling, and output format.  
   - Attach OpenAI GPT-4 and Anthropic Claude API credentials.  
   - Connect Simple Memory node for context.  
   - Add Brave Search (multiple nodes), Brave News, and Blog Content tool nodes for data sources.

5. **Add Brave Search Nodes:**  
   - Multiple instances with dynamic query input from AI expressions.  
   - Connect to Research Agent and other agents as per research phases.  
   - Configure Brave Search API credentials.  
   - Ensure sequential execution with 2-second enforced delays in agent logic.

6. **Add Brave News Node:**  
   - Configure dynamic query input for recent news.  
   - Connect to Research Agent.

7. **Add Blog Content tool Nodes:**  
   - Type: Ghost Content API Node  
   - Configure to fetch all existing blog content in plaintext.  
   - Connect to Research Agent, Content Generator Agent, and Content Editor Agent for internal content reference.

8. **Add Content Generation Agent Node:**  
   - Type: LangChain Agent Tool  
   - Connect input from Blog Content Orchestrator Agent.  
   - Configure system prompt for section-by-section research, content writing, citation integration, internal linking, and delay compliance.  
   - Attach OpenAI and Anthropic API credentials.  
   - Connect Simple Memory4.

9. **Add Content Editor Agent Node:**  
   - Type: LangChain Agent Tool  
   - Connect input from Blog Content Orchestrator Agent.  
   - Configure system prompt for editing workflow, citation verification, readability improvements, HTML formatting, and quality assurance.  
   - Attach OpenAI and Anthropic API credentials.  
   - Connect Simple Memory node for context.

10. **Add SEO Optimizer Agent Node:**  
    - Type: LangChain Agent Tool  
    - Connect input from Blog Content Orchestrator Agent.  
    - Configure system prompt for keyword research, SEO on-page optimization, technical SEO, competitor analysis, and delay enforcement.  
    - Attach OpenAI and Anthropic API credentials.  
    - Connect Simple Memory.

11. **Add Blog HTML Publisher Agent Node:**  
    - Type: LangChain Agent Tool  
    - Connect input from Blog Content Orchestrator Agent.  
    - Configure system prompt for HTML generation, meta tags, schema, social media tags, and Ghost CMS publishing.  
    - Attach OpenAI and Anthropic API credentials.  
    - Connect Simple Memory2.

12. **Add Ghost Publisher Tool Node:**  
    - Type: Ghost CMS Admin API Node  
    - Connect input from Blog HTML Publisher Agent.  
    - Configure operation "create" with parameters: title, content, slug, status (draft), meta_title, meta_description, and canonical_url dynamically injected from AI expressions.  
    - Attach Ghost Admin API credentials.

13. **Add Additional Simple Memory Nodes:**  
    - For SEO Optimizer Agent, Content Generator Agent, Content Editor Agent, and HTML Publisher Agent as needed.  
    - Configure default context window.

14. **Interconnect Nodes:**  
    - Ensure outputs and inputs flow logically:  
      - User input → Orchestrator → Research Agent → TOC approval → Content Generator → SEO Optimizer → Content Editor → HTML Publisher → Ghost Publisher Tool.  
    - Connect Brave Search and Brave News nodes as AI tool inputs to relevant agents.  
    - Connect Ghost Content tool nodes appropriately for internal content reference.

15. **Set Credentials:**  
    - OpenAI API key for GPT-4 models.  
    - Anthropic API key for Claude models.  
    - Brave Search API key for search nodes.  
    - Ghost CMS Admin API key for publishing node.  
    - Ghost Content API key for content retrieval nodes.

16. **Configure Error Handling and Retry Policies:**  
    - Enable retry on fail for agents with recommended backoff delays.  
    - Implement validation checks for required fields before passing to next agent.  
    - Enforce minimum 2-second delays between Brave Search calls via agent prompt logic.

17. **Add Sticky Notes:**  
    - Add descriptive sticky notes to document roles of key agents and overall workflow for maintainability.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      | Context or Link                                                                                                         |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------|
| **Complete Workflow Overview:** This workflow automates the full blogging process from topic input to publication, leveraging multiple AI agents for research, writing, SEO, editing, and publishing. Requires API keys for OpenAI, Anthropic, Brave Search, and Ghost CMS.                                                                                                                                                                                                                                                                                                                                                                                        | See Sticky Note6 content                                                                                               |
| **Research Agent:** Conducts comprehensive research using Brave Search, Brave News, and existing blog content, enforcing strict delays to respect rate limits. Generates a structured table of contents and research insights.                                                                                                                                                                                                                                                                                                                                                                                              | See Sticky Note1 content                                                                                               |
| **Content Generator Agent:** Writes detailed, well-cited blog sections based on approved TOC and research, maintaining style consistency and integrating internal and external links.                                                                                                                                                                                                                                                                                                                                                                                                                    | See Sticky Note2 content                                                                                               |
| **SEO Optimizer Agent:** Performs keyword research, on-page SEO, technical SEO, and optimization recommendations to maximize search visibility.                                                                                                                                                                                                                                                                                                                                                                                                                                                       | See Sticky Note3 content                                                                                               |
| **Content Editor Agent:** Enhances readability, verifies citations, improves flow, and produces publication-ready HTML content. Handles citation research with enforced delays.                                                                                                                                                                                                                                                                                                                                                                                                                      | See Sticky Note4 content                                                                                               |
| **Blog HTML Publisher Agent:** Generates SEO-optimized HTML, complete with meta tags and schema markup. Publishes content as draft to Ghost CMS using Ghost Publisher Tool. Execution of the publishing step is mandatory.                                                                                                                                                                                                                                                                                                                                                                             | See Sticky Note5 content                                                                                               |
| **Brave Search Delay Compliance:** All agents using Brave Search enforce minimum 2-second delays between API calls, with retry and backoff logic for rate limits. This is critical for reliable operation.                                                                                                                                                                                                                                                                                                                                                                                             | Documented in multiple agent prompts                                                                                   |
| **User Approval Gates:** The workflow requires explicit user approval after TOC creation and final draft editing before proceeding to next phases, ensuring quality control and user satisfaction.                                                                                                                                                                                                                                                                                                                                                                                                   | Detailed in Blog Content Orchestrator Agent prompt                                                                    |
| **Error Handling:** Includes retry policies, validation of required fields, and clear fallback instructions for failed API calls or missing data. Logs all inter-agent communications for troubleshooting.                                                                                                                                                                                                                                                                                                                                                                                             | Detailed in agent configuration and orchestrator instructions                                                         |

---

**Disclaimer:** The text documented above is derived exclusively from an automated n8n workflow using AI agents for content creation and publishing. It complies fully with content policies and contains no illegal or offensive material. All processed data is legal and public.