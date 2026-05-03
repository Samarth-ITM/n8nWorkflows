Research AI Agent Team with auto citations using OpenRouter and Perplexity

https://n8nworkflows.xyz/workflows/research-ai-agent-team-with-auto-citations-using-openrouter-and-perplexity-2607


# Research AI Agent Team with auto citations using OpenRouter and Perplexity

### 1. Workflow Overview

This workflow is an AI-powered automated article generator designed to produce comprehensive, well-structured articles on any user-defined topic. It orchestrates a multi-agent AI team to streamline the research, planning, writing, and editing process, ensuring the final output includes credible citations and a coherent narrative.

The workflow logically divides into the following functional blocks:

- **1.1 Input Reception:** Captures user inputs via a form trigger, defining the topic, tone, word count, and other customizable parameters.
- **1.2 Research Leader & Planning:** The Research Leader agent initiates the research by generating a table of contents, which the Project Planner then breaks down into detailed sections.
- **1.3 Research Assistants:** Multiple assistant agents conduct deep research on assigned sections using internet search capabilities powered by Perplexity AI.
- **1.4 Content Assembly & Editing:** Merge the researched chapter titles and texts, refine and compile the article via an Editor agent, create a final article title, and publish it.
- **1.5 Perplexity API & Citation Handling:** Integrates Perplexity AI API calls for real-time internet search and citation extraction, ensuring content credibility.
- **1.6 Output & Publication:** Final article output preparation and publishing to a Ghost CMS.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:** Captures initial user parameters for article generation.
- **Nodes Involved:**  
  - `n8n Form Trigger`
- **Node Details:**

  - **n8n Form Trigger**  
    - *Type:* Form Trigger  
    - *Role:* Entry point capturing user inputs such as topic, tone, word count, and number of sections.  
    - *Configuration:* Configured with a webhook to receive form submissions.  
    - *Expressions/Variables:* User responses passed as input data to downstream nodes.  
    - *Input:* External user form POST request  
    - *Output:* Data sent to the Research Leader agent node.  
    - *Version:* 2.1  
    - *Edge cases:* Missing or malformed input could cause downstream failures; no explicit validation shown. Authentication depends on webhook security.

#### 2.2 Research Leader & Planning

- **Overview:** The Research Leader agent generates the initial table of contents, then the Project Planner agent decomposes it into detailed sections for research.
- **Nodes Involved:**  
  - `Research Leader 🔬`  
  - `Project Planner`  
  - `Perplexity_tool2` (tool workflow for Project Planner)  
  - `Structured Output Parser`  
  - `OpenAI Chat Model2` (LM for Research Leader)  
  - `OpenAI Chat Model` (LM for Project Planner)  
  - `Perplexity_tool` (tool workflow for Research Leader)  
- **Node Details:**

  - **Research Leader 🔬**  
    - *Type:* LangChain Agent  
    - *Role:* Initiates research, plans the article by creating a table of contents from user input.  
    - *Configuration:* Uses OpenAI Chat Model2 as its language model, and Perplexity_tool as its AI tool for internet research.  
    - *Input:* Data from Form Trigger.  
    - *Output:* Table of contents to Project Planner.  
    - *Version:* 1.6  
    - *Edge cases:* API failures (OpenAI or Perplexity), incomplete or vague inputs may lead to poor content planning.

  - **Project Planner**  
    - *Type:* LangChain Agent  
    - *Role:* Breaks down the table of contents into detailed, manageable sections for research assistants.  
    - *Configuration:* Uses OpenAI Chat Model as its language model, Perplexity_tool2 as its AI tool. Output is structured using Structured Output Parser.  
    - *Input:* Table of contents from Research Leader.  
    - *Output:* List of sections sent to Delegate to Research Assistants node.  
    - *Version:* 1.7  
    - *Edge cases:* Parsing errors in structured output, API timeouts.

  - **Structured Output Parser**  
    - *Type:* LangChain Structured Output Parser  
    - *Role:* Parses the structured response from Project Planner into usable JSON or structured data.  
    - *Input:* Raw output from Project Planner.  
    - *Output:* Structured data to Project Planner for further routing.  
    - *Version:* 1.2  
    - *Edge cases:* Parsing errors if output deviates from expected schema.

  - **Perplexity_tool & Perplexity_tool2**  
    - *Type:* LangChain tool workflows  
    - *Role:* Provide internet search and citation capabilities to Research Leader and Project Planner, respectively, leveraging Perplexity AI API.  
    - *Input:* Query prompts from respective agents.  
    - *Output:* Search results with citations.  
    - *Version:* 1.2  
    - *Edge cases:* API key invalid, rate limiting, or network errors.

  - **OpenAI Chat Model2 & OpenAI Chat Model**  
    - *Type:* LangChain OpenAI Chat Language Model  
    - *Role:* Core language models for the Research Leader and Project Planner agents.  
    - *Configuration:* Presumably configured with OpenAI API credentials.  
    - *Version:* 1  
    - *Edge cases:* API authentication issues, quota exhaustion.

#### 2.3 Research Assistants

- **Overview:** Multiple AI agents perform in-depth research on each section assigned by the Project Planner, generating chapter content with citations.
- **Nodes Involved:**  
  - `Delegate to Research Assistants`  
  - `Research Assistant`  
  - `Perplexity_tool1` (tool workflow for Research Assistant)  
  - `OpenAI Chat Model3` (LM for Research Assistant)  
  - `Merge chapters title and text`  
- **Node Details:**

  - **Delegate to Research Assistants**  
    - *Type:* Split Out  
    - *Role:* Splits the sections list into individual tasks for Research Assistant agents and merges chapter titles and texts afterward.  
    - *Input:* Sections list from Project Planner.  
    - *Output:* Parallel branches to Research Assistant and merge node.  
    - *Version:* 1  
    - *Edge cases:* Improper splitting or data format mismatch.

  - **Research Assistant**  
    - *Type:* LangChain Agent  
    - *Role:* Conducts detailed research on assigned sections using Perplexity and OpenAI LLM to generate content with citations.  
    - *Configuration:* Uses OpenAI Chat Model3 and Perplexity_tool1. Retry on failure enabled for robustness.  
    - *Input:* Section details from Delegate node.  
    - *Output:* Chapter titles and texts.  
    - *Version:* 1.6  
    - *Edge cases:* API errors, incomplete research, or timeout.

  - **Merge chapters title and text**  
    - *Type:* Merge  
    - *Role:* Combines individual chapter titles and texts into a single dataset for final article assembly.  
    - *Input:* Outputs from multiple Research Assistant executions.  
    - *Output:* Combined article content to Final article text node.  
    - *Version:* 2.1  
    - *Edge cases:* Mismatch in data structure or incomplete merges.

  - **Perplexity_tool1 & OpenAI Chat Model3**  
    - *Roles and configuration similar to previous tool and LM nodes but specialized for Research Assistant tasks.*

#### 2.4 Content Assembly & Editing

- **Overview:** Compiles and refines the merged research content into a polished article, generates the final article title, and publishes it.
- **Nodes Involved:**  
  - `Final article text`  
  - `Editor`  
  - `Create title`  
  - `Ghost`  
  - `OpenAI Chat Model1` (LM for Editor and Title creator)  
- **Node Details:**

  - **Final article text**  
    - *Type:* Code  
    - *Role:* Custom code node to prepare or format the merged chapter content for editing.  
    - *Input:* Merged chapters from Merge node.  
    - *Output:* Cleaned article content to Editor.  
    - *Version:* 1  
    - *Edge cases:* Code errors or unexpected input data format.

  - **Editor**  
    - *Type:* LangChain Agent  
    - *Role:* Edits and compiles the final article for coherence, style, and citation formatting.  
    - *Configuration:* Uses OpenAI Chat Model1 as language model; input from Final article text node.  
    - *Output:* Final polished article text to Create title node.  
    - *Version:* 1.6  
    - *Edge cases:* Model errors or API failures.

  - **Create title**  
    - *Type:* LangChain Chain LLM  
    - *Role:* Generates a compelling title for the article based on the edited content.  
    - *Input:* Final article from Editor.  
    - *Output:* Title sent to Ghost CMS for publication.  
    - *Version:* 1.4  
    - *Edge cases:* Title relevance or creativity limitations.

  - **Ghost**  
    - *Type:* Ghost CMS node  
    - *Role:* Publishes the finalized article with title to the Ghost blogging platform.  
    - *Input:* Title and article content.  
    - *Output:* Published article confirmation.  
    - *Version:* 1  
    - *Edge cases:* Authentication errors, CMS API failures, network issues.

  - **OpenAI Chat Model1**  
    - *Used for both Editor and Create title nodes, configured with OpenAI credentials.*

#### 2.5 Perplexity API & Response Handling

- **Overview:** Handles direct calls to Perplexity API for search queries and processes responses for integration into the workflow.
- **Nodes Involved:**  
  - `Execute Workflow Trigger`  
  - `Perplexity API`  
  - `Response`  
- **Node Details:**

  - **Execute Workflow Trigger**  
    - *Type:* Execute Workflow Trigger  
    - *Role:* Triggers workflows that call the Perplexity API directly.  
    - *Output:* Passes data to Perplexity API node.  
    - *Version:* 1

  - **Perplexity API**  
    - *Type:* HTTP Request  
    - *Role:* Makes HTTP requests to the Perplexity API endpoint to retrieve search results and citations.  
    - *Configuration:* Requires API key in headers (Authorization: Bearer <api-key>).  
    - *Output:* Raw API response to Response node.  
    - *Version:* 4.2  
    - *Edge cases:* Invalid API key, rate limit, network errors.

  - **Response**  
    - *Type:* Set  
    - *Role:* Processes and formats the API response for downstream consumption.  
    - *Version:* 3.4

---

### 3. Summary Table

| Node Name                   | Node Type                          | Functional Role                         | Input Node(s)                | Output Node(s)                 | Sticky Note                                            |
|-----------------------------|----------------------------------|---------------------------------------|-----------------------------|-------------------------------|--------------------------------------------------------|
| n8n Form Trigger            | Form Trigger                     | User input reception                   | External                    | Research Leader 🔬              |                                                        |
| Research Leader 🔬          | LangChain Agent                  | Initial research, TOC generation       | n8n Form Trigger            | Project Planner                |                                                        |
| OpenAI Chat Model2          | LangChain OpenAI Chat Model      | Language model for Research Leader     | —                           | Research Leader 🔬             |                                                        |
| Perplexity_tool             | LangChain Tool Workflow          | Internet search tool for Research Leader | —                           | Research Leader 🔬             |                                                        |
| Project Planner             | LangChain Agent                  | Breakdown of TOC into sections         | Research Leader 🔬           | Delegate to Research Assistants |                                                        |
| OpenAI Chat Model           | LangChain OpenAI Chat Model      | Language model for Project Planner     | —                           | Project Planner               |                                                        |
| Structured Output Parser    | LangChain Output Parser Structured | Parses Project Planner output          | Project Planner             | Project Planner               |                                                        |
| Perplexity_tool2            | LangChain Tool Workflow          | Internet search tool for Project Planner | —                           | Project Planner               |                                                        |
| Delegate to Research Assistants | Split Out                     | Splits sections for Research Assistants | Project Planner             | Research Assistant, Merge chapters title and text |                                                        |
| Research Assistant          | LangChain Agent                  | Detailed research on assigned sections | Delegate to Research Assistants | Merge chapters title and text |                                                        |
| OpenAI Chat Model3          | LangChain OpenAI Chat Model      | Language model for Research Assistant  | —                           | Research Assistant            |                                                        |
| Perplexity_tool1            | LangChain Tool Workflow          | Internet search tool for Research Assistant | —                           | Research Assistant            |                                                        |
| Merge chapters title and text | Merge                          | Combines research chapters             | Delegate to Research Assistants, Research Assistant | Final article text           |                                                        |
| Final article text          | Code                            | Formats merged chapters for editing    | Merge chapters title and text | Editor                       |                                                        |
| Editor                      | LangChain Agent                  | Edits and compiles final article       | Final article text          | Create title                  |                                                        |
| OpenAI Chat Model1          | LangChain OpenAI Chat Model      | Language model for Editor and Title creation | —                           | Editor, Create title          |                                                        |
| Create title                | LangChain Chain LLM              | Generates final article title           | Editor                      | Ghost                        |                                                        |
| Ghost                      | Ghost CMS                       | Publishes final article                 | Create title                | —                           |                                                        |
| Execute Workflow Trigger    | Execute Workflow Trigger         | Triggers Perplexity API calls           | External                    | Perplexity API                |                                                        |
| Perplexity API              | HTTP Request                    | Calls Perplexity API for searches       | Execute Workflow Trigger    | Response                     | Credential required: Authorization header with Bearer token |
| Response                   | Set                             | Processes API response                   | Perplexity API              | —                           |                                                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Form Trigger node**  
   - Type: Form Trigger  
   - Configure webhook for receiving user inputs (topic, tone, word count, number of sections).  
   - Position it as the workflow entry point.

2. **Add a LangChain OpenAI Chat Model node (OpenAI Chat Model2)**  
   - Configure with OpenAI credentials.  
   - Assign as language model for Research Leader agent.

3. **Add a LangChain Tool Workflow node (Perplexity_tool)**  
   - Configure to call Perplexity AI API with API key in Authorization header.  
   - Credential should be set as `Authorization: Bearer <api-key>`.  
   - This node serves as an AI tool for the Research Leader.

4. **Add a LangChain Agent node (Research Leader 🔬)**  
   - Assign OpenAI Chat Model2 as LM and Perplexity_tool as AI tool.  
   - Connect input from Form Trigger.  
   - Output to Project Planner node.

5. **Add a LangChain OpenAI Chat Model node (OpenAI Chat Model)**  
   - Configure with OpenAI credentials for Project Planner.

6. **Add a LangChain Tool Workflow node (Perplexity_tool2)**  
   - Similar configuration as Perplexity_tool for Project Planner.

7. **Add a LangChain Agent node (Project Planner)**  
   - Assign OpenAI Chat Model and Perplexity_tool2.  
   - Connect input from Research Leader.  
   - Output to Structured Output Parser.

8. **Add a LangChain Structured Output Parser node**  
   - Configure expected output format to parse Project Planner's response.  
   - Connect output back to Project Planner or Delegate to Research Assistants.

9. **Add a Split Out node (Delegate to Research Assistants)**  
   - Input: Parsed Project Planner output containing sections.  
   - Outputs: One branch to Research Assistant nodes, another to Merge node.

10. **Add a LangChain OpenAI Chat Model node (OpenAI Chat Model3)**  
    - Configure with OpenAI credentials for Research Assistants.

11. **Add a LangChain Tool Workflow node (Perplexity_tool1)**  
    - Configure with Perplexity API credentials for Research Assistants.

12. **Add a LangChain Agent node (Research Assistant)**  
    - Assign OpenAI Chat Model3 and Perplexity_tool1.  
    - Enable retry on failure for robustness.  
    - Connect input from Delegate to Research Assistants.

13. **Add a Merge node (Merge chapters title and text)**  
    - Merge outputs from multiple Research Assistant executions.  
    - Connect input from Delegate to Research Assistants and Research Assistant outputs.

14. **Add a Code node (Final article text)**  
    - Write custom code to format and prepare merged chapters for editing.  
    - Connect input from Merge node.

15. **Add a LangChain OpenAI Chat Model node (OpenAI Chat Model1)**  
    - Configure with OpenAI credentials for Editor and Title creation.

16. **Add a LangChain Agent node (Editor)**  
    - Assign OpenAI Chat Model1 as LM.  
    - Connect input from Final article text.  
    - Output to Create title node.

17. **Add a LangChain Chain LLM node (Create title)**  
    - Assign OpenAI Chat Model1.  
    - Connect input from Editor.  
    - Output to Ghost CMS.

18. **Add a Ghost node**  
    - Configure with Ghost CMS credentials/API key.  
    - Connect input from Create title node for publishing.

19. **Add HTTP Request node (Perplexity API)**  
    - Configure to call Perplexity API directly.  
    - Add Authorization header with Bearer token.  
    - Input from Execute Workflow Trigger node.

20. **Add Execute Workflow Trigger node**  
    - Use to trigger Perplexity API calls as needed.

21. **Add Set node (Response)**  
    - Process and format Perplexity API responses.

22. **Connect all nodes according to logical flow described in section 1 and 2.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                 | Context or Link                                                      |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------|
| To use Perplexity API, create an account at OpenRouter.ai and obtain an API key. Use this key in the Authorization header as `Bearer <api-key>`.             | Setup instructions, API key configuration in Perplexity API node  |
| The workflow implements a multi-agent AI team model for efficient and modular content creation with automatic citation integration.                         | Conceptual design                                                  |
| For publishing, Ghost CMS integration requires proper API key and setup on the Ghost platform.                                                              | Ghost CMS documentation                                            |
| Sticky notes in the workflow provide contextual comments for node groups and overall workflow guidance.                                                     | Visible in workflow editor                                         |

---

This document provides a detailed, structured reference for understanding, reproducing, and maintaining the "Research AI Agent Team with auto citations using OpenRouter and Perplexity" n8n workflow.