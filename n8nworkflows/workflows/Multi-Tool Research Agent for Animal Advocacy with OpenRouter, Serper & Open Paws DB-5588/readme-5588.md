Multi-Tool Research Agent for Animal Advocacy with OpenRouter, Serper & Open Paws DB

https://n8nworkflows.xyz/workflows/multi-tool-research-agent-for-animal-advocacy-with-openrouter--serper---open-paws-db-5588


# Multi-Tool Research Agent for Animal Advocacy with OpenRouter, Serper & Open Paws DB

### 1. Workflow Overview

This n8n workflow, titled **Multi-Tool Research Agent for Animal Advocacy with OpenRouter, Serper & Open Paws DB**, serves as a comprehensive autonomous research and content analysis agent tailored for animal advocacy. It is designed to intake a user query (chat input) and autonomously apply multiple specialized tools and data sources to generate a well-researched, evidence-backed, and high-quality response without user intervention.

The workflow is structured into these logical blocks:

- **1.1 Input Reception**: Captures chat messages or external workflow calls and extracts necessary input fields.
- **1.2 Memory Management**: Maintains conversational/contextual memory across sessions.
- **1.3 AI Agent Processing**: Core agent logic invoking language models and orchestrating tool usage based on the input.
- **1.4 External Tool Integration**: Calls multiple APIs and services for live data retrieval, scraping, and database querying.
- **1.5 Decision and Response Handling**: Checks for empty or incomplete AI outputs and applies fallback mechanisms.
- **1.6 Text Scoring Sub-Workflow**: Evaluates the quality and advocacy alignment of generated or retrieved text content.
- **1.7 Output Preparation**: Structures the final output including the main answer and intermediate reasoning steps.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
Receives chat input either via a chat trigger webhook or when called by another workflow. It extracts and sets key fields like `chatInput` and `sessionId` for downstream processing.

**Nodes Involved:**  
- When chat message received  
- When Executed by Another Workflow  
- Set Fields

**Node Details:**

- **When chat message received**  
  - Type: `@n8n/n8n-nodes-langchain.chatTrigger`  
  - Role: Listens for incoming chat messages over webhook  
  - Configuration: Default webhook, no additional parameters  
  - Inputs: External webhook calls  
  - Outputs: Passes raw input JSON downstream  
  - Edge Cases: Missing or malformed webhook payloads, webhook downtime

- **When Executed by Another Workflow**  
  - Type: `n8n-nodes-base.executeWorkflowTrigger`  
  - Role: Allows this workflow to be triggered programmatically by other workflows  
  - Parameters: Accepts inputs `chatInput` and `sessionId`  
  - Inputs: Internal workflow calls  
  - Outputs: Passes inputs downstream  
  - Edge Cases: Incorrect input parameters, failed upstream workflow call

- **Set Fields**  
  - Type: `n8n-nodes-base.set`  
  - Role: Maps raw input JSON fields to standardized variables `chatInput` and `sessionId`  
  - Configuration: Stores `chatInput` and `sessionId` as string variables  
  - Inputs: Output from either trigger node  
  - Outputs: Data with standardized field names for AI Agent  
  - Edge Cases: Missing fields in input JSON causing null or empty variables

---

#### 2.2 Memory Management

**Overview:**  
Manages session-specific memory to retain context across multiple interactions, enabling the AI agent to maintain continuity.

**Nodes Involved:**  
- Simple Memory

**Node Details:**

- **Simple Memory**  
  - Type: `@n8n/n8n-nodes-langchain.memoryBufferWindow`  
  - Role: Stores and retrieves a sliding window of recent conversation context keyed by sessionId  
  - Configuration: Uses `sessionId` as a custom session key; retains last 10 context entries  
  - Inputs: None directly; connected to AI Agent’s memory interface  
  - Outputs: Provides contextual memory to AI Agent  
  - Edge Cases: Session key missing or invalid, memory overflow or truncation, session collisions

---

#### 2.3 AI Agent Processing

**Overview:**  
Central agent node coordinating the reasoning, tool usage, and response generation using OpenRouter language model and multiple integrated tools. It follows strict execution directives to autonomously complete the task.

**Nodes Involved:**  
- AI Agent  
- OpenRouter Chat Model  
- OpenRouter Chat Model1

**Node Details:**

- **AI Agent**  
  - Type: `@n8n/n8n-nodes-langchain.agent`  
  - Role: Main orchestrator executing the agent's instructions and tool invocations  
  - Configuration:  
    - Uses input text from `Set Fields` node (`chatInput`)  
    - Max iterations: 100 to allow multiple tool calls and reasoning cycles  
    - System message: Detailed prompt enforcing autonomous operation, tool usage policies, error recovery, and completion criteria  
    - Returns intermediate steps for transparency  
  - Inputs: `chatInput`, memory from Simple Memory, and tool outputs  
  - Outputs: Produces final answer or intermediate results  
  - Edge Cases: Tool failures, empty responses, infinite loops (max iterations safeguard), expression evaluation errors

- **OpenRouter Chat Model**  
  - Type: `@n8n/n8n-nodes-langchain.lmChatOpenRouter`  
  - Role: Provides LLM completions for the AI Agent  
  - Configuration: Uses model `google/gemini-2.5-flash-preview` via OpenRouter API  
  - Inputs: Prompts from AI Agent  
  - Outputs: Generated text responses  
  - Edge Cases: API quota exceeded, network timeouts, model unavailability

- **OpenRouter Chat Model1**  
  - Type: `@n8n/n8n-nodes-langchain.lmChatOpenRouter`  
  - Role: Secondary LLM node used in fallback chain to fix empty responses  
  - Configuration: Same model and credentials as above  
  - Inputs: Prompt generated by Fix Empty Response node  
  - Outputs: Refined text output if original is empty  
  - Edge Cases: Same as above

---

#### 2.4 External Tool Integration

**Overview:**  
Multiple HTTP request nodes enable live data retrieval and scraping from external APIs to provide current and domain-specific information.

**Nodes Involved:**  
- Serper API  
- Database Retrieval  
- Web Scraper Tool  
- Twitter Post Scraper  
- Twitter Profile Scraper  
- Instagram Profile Scraper  
- Linkedin Person and Company Scraper  
- Email Finder  
- Email Verifier

**Node Details:**

- **Serper API**  
  - Type: `@n8n/n8n-nodes-langchain.toolHttpRequest`  
  - Role: Performs live Google searches via Serper API with multiple endpoints (search, news, patents, maps, etc.)  
  - Configuration: URL with endpoint placeholder `{endpoint}`, HTTP header auth with API key  
  - Inputs: Query parameters including search query, location, time filters  
  - Outputs: Search results including snippets, FAQs, organic results  
  - Edge Cases: Invalid API key, rate limits, malformed queries

- **Database Retrieval**  
  - Type: `@n8n/n8n-nodes-langchain.toolHttpRequest`  
  - Role: Queries the Open Paws knowledge base using GraphQL hybrid search  
  - Configuration: POST request with GraphQL query embedding user search  
  - Authentication: HTTP Custom Auth with authorization headers (Bearer Open Paws key and OpenAI API key)  
  - Inputs: User query embedded in GraphQL query  
  - Outputs: Up to 5 relevant content entries with summaries and metadata  
  - Edge Cases: Auth failures, malformed GraphQL, empty results

- **Web Scraper Tool**  
  - Type: `@n8n/n8n-nodes-langchain.toolHttpRequest`  
  - Role: Extracts clean textual content from webpages via Jina AI service  
  - Configuration: URL placeholder `{url}`, HTTP header auth with Jina API key  
  - Inputs: Website URL to scrape  
  - Outputs: Extracted readable text from webpage  
  - Edge Cases: Invalid URLs, page inaccessible, rate limits

- **Twitter Post Scraper**  
  - Type: `@n8n/n8n-nodes-langchain.toolHttpRequest`  
  - Role: Retrieves content of specific tweets via Scrapingdog API  
  - Configuration: URL with `{tweetId}` placeholder, HTTP Query Auth with Scrapingdog API key  
  - Inputs: Tweet ID  
  - Outputs: Parsed tweet content  
  - Edge Cases: Invalid ID, private/protected tweets, rate limits

- **Twitter Profile Scraper**  
  - Type: `@n8n/n8n-nodes-langchain.toolHttpRequest`  
  - Role: Scrapes tweets and profile info from Twitter profiles via Scrapingdog  
  - Configuration: URL with `{profileId}`, HTTP Query Auth  
  - Inputs: Twitter username  
  - Outputs: List of tweets and profile metadata  
  - Edge Cases: Private profiles, rate limits

- **Instagram Profile Scraper**  
  - Type: `@n8n/n8n-nodes-langchain.toolHttpRequest`  
  - Role: Scrapes public Instagram posts including captions and images  
  - Configuration: URL with `{username}`, HTTP Query Auth  
  - Inputs: Instagram profile username  
  - Outputs: Post data and metadata  
  - Edge Cases: Private profiles, API access limits

- **Linkedin Person and Company Scraper**  
  - Type: `@n8n/n8n-nodes-langchain.toolHttpRequest`  
  - Role: Retrieves public LinkedIn profile or company info via Scrapingdog  
  - Configuration: URL with `{linkedinID}`, HTTP Query Auth  
  - Inputs: LinkedIn profile or company identifier  
  - Outputs: Profile details including name, title, summary  
  - Edge Cases: Private profiles, invalid IDs, rate limits

- **Email Finder**  
  - Type: `n8n-nodes-base.hunterTool`  
  - Role: Finds likely email addresses based on domain, first and last name using Hunter.io API  
  - Configuration: Inputs from AI-generated variables (domain, firstname, lastname)  
  - Credentials: Hunter API native credential  
  - Outputs: Candidate email addresses  
  - Edge Cases: No email found, API quota exceeded

- **Email Verifier**  
  - Type: `n8n-nodes-base.hunterTool`  
  - Role: Verifies email deliverability and validity  
  - Configuration: Inputs email from AI-generated field  
  - Credentials: Hunter API  
  - Outputs: Verification status and metadata  
  - Edge Cases: Invalid email format, API failures

---

#### 2.5 Decision and Response Handling

**Overview:**  
Evaluates if AI agent’s output is empty or not; if empty, triggers fallback LLM completion; otherwise, prepares final output.

**Nodes Involved:**  
- If  
- Fix Empty Response  
- Set Output (If Empty)  
- Set Output (If Not Empty)  
- OpenRouter Chat Model1

**Node Details:**

- **If**  
  - Type: `n8n-nodes-base.if`  
  - Role: Checks if `output` field from AI Agent is empty string  
  - Inputs: AI Agent output JSON  
  - Outputs: Two branches: empty or non-empty output  
  - Edge Cases: Unexpected data types, missing fields

- **Fix Empty Response**  
  - Type: `@n8n/n8n-nodes-langchain.chainLlm`  
  - Role: Uses an LLM chain to generate a fallback answer when AI Agent output is empty  
  - Configuration: Uses prompt combining user query and intermediate steps for regeneration  
  - Inputs: Original user query, intermediate reasoning logs  
  - Outputs: Regenerated text output  
  - Edge Cases: LLM failure, empty regeneration

- **Set Output (If Empty)**  
  - Type: `n8n-nodes-base.set`  
  - Role: Stores fallback output text and intermediate steps for final result  
  - Inputs: Output from Fix Empty Response node  
  - Outputs: Final output JSON  
  - Edge Cases: None

- **Set Output (If Not Empty)**  
  - Type: `n8n-nodes-base.set`  
  - Role: Stores AI Agent’s original output and intermediate steps for final result  
  - Inputs: Output from AI Agent node  
  - Outputs: Final output JSON  
  - Edge Cases: None

- **OpenRouter Chat Model1**  
  - (Also part of fallback chain as described above)

---

#### 2.6 Text Scoring Sub-Workflow

**Overview:**  
Evaluates the predicted online performance and advocate preference of pieces of text to inform or refine content.

**Nodes Involved:**  
- Score Text (sub-workflow node)

**Node Details:**

- **Score Text**  
  - Type: `@n8n/n8n-nodes-langchain.toolWorkflow`  
  - Role: Invokes external sub-workflow that runs regression models for text scoring  
  - Configuration: Inputs text to be scored; outputs two scores—Performance and Advocate Preference (0 to 1 scale)  
  - Inputs: Text from AI or retrieved content  
  - Outputs: Scores for content evaluation  
  - Edge Cases: Sub-workflow unavailability, invalid input text

---

#### 2.7 Output Preparation

**Overview:**  
Prepares the final JSON output including the main answer and the internal intermediate steps for transparency and traceability.

**Nodes Involved:**  
- Set Output (If Empty)  
- Set Output (If Not Empty)

**Node Details:**  
(Described above in Decision and Response Handling)

---

### 3. Summary Table

| Node Name                       | Node Type                                         | Functional Role                                      | Input Node(s)                     | Output Node(s)                | Sticky Note                                                                                                                             |
|--------------------------------|--------------------------------------------------|-----------------------------------------------------|----------------------------------|------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------|
| When chat message received      | @n8n/n8n-nodes-langchain.chatTrigger             | Entry point for chat input                           | (External webhook)                | Set Fields                   |                                                                                                                                         |
| When Executed by Another Workflow| n8n-nodes-base.executeWorkflowTrigger            | Entry point when called by other workflows          | (External workflow call)          | Set Fields                   |                                                                                                                                         |
| Set Fields                     | n8n-nodes-base.set                                | Maps input JSON fields to `chatInput` and `sessionId`| When chat message received, When Executed by Another Workflow | AI Agent                    |                                                                                                                                         |
| Simple Memory                 | @n8n/n8n-nodes-langchain.memoryBufferWindow       | Maintains session memory/context                      | -                                | AI Agent                    |                                                                                                                                         |
| AI Agent                      | @n8n/n8n-nodes-langchain.agent                     | Core autonomous agent orchestrating tool usage       | Set Fields, Simple Memory, Tool nodes | If                        |                                                                                                                                         |
| OpenRouter Chat Model         | @n8n/n8n-nodes-langchain.lmChatOpenRouter          | LLM for AI Agent completions                          | AI Agent                        | AI Agent                    |                                                                                                                                         |
| Serper API                   | @n8n/n8n-nodes-langchain.toolHttpRequest           | Live Google search and info retrieval                 | AI Agent                        | AI Agent                    |                                                                                                                                         |
| Database Retrieval            | @n8n/n8n-nodes-langchain.toolHttpRequest           | Query Open Paws knowledge base                        | AI Agent                        | AI Agent                    |                                                                                                                                         |
| Web Scraper Tool              | @n8n/n8n-nodes-langchain.toolHttpRequest           | Extract text from webpages                            | AI Agent                        | AI Agent                    |                                                                                                                                         |
| Twitter Post Scraper          | @n8n/n8n-nodes-langchain.toolHttpRequest           | Scrape specific Twitter posts                         | AI Agent                        | AI Agent                    |                                                                                                                                         |
| Twitter Profile Scraper       | @n8n/n8n-nodes-langchain.toolHttpRequest           | Scrape Twitter profile content                        | AI Agent                        | AI Agent                    |                                                                                                                                         |
| Instagram Profile Scraper     | @n8n/n8n-nodes-langchain.toolHttpRequest           | Scrape Instagram profile posts                        | AI Agent                        | AI Agent                    |                                                                                                                                         |
| Linkedin Person and Company Scraper | @n8n/n8n-nodes-langchain.toolHttpRequest       | Scrape LinkedIn profile or company info               | AI Agent                        | AI Agent                    |                                                                                                                                         |
| Email Finder                 | n8n-nodes-base.hunterTool                           | Find email addresses                                  | AI Agent                        | AI Agent                    |                                                                                                                                         |
| Email Verifier               | n8n-nodes-base.hunterTool                           | Verify email addresses                                | AI Agent                        | AI Agent                    |                                                                                                                                         |
| If                          | n8n-nodes-base.if                                   | Check if AI Agent output is empty                     | AI Agent                        | Fix Empty Response, Set Output (If Not Empty) |                                                                                                                                         |
| Fix Empty Response           | @n8n/n8n-nodes-langchain.chainLlm                   | Regenerate output if AI Agent output is empty        | If                            | Set Output (If Empty)        |                                                                                                                                         |
| Set Output (If Empty)        | n8n-nodes-base.set                                  | Store fallback output and intermediate steps         | Fix Empty Response              | (Workflow output)            |                                                                                                                                         |
| Set Output (If Not Empty)    | n8n-nodes-base.set                                  | Store AI Agent output and intermediate steps         | If                            | (Workflow output)            |                                                                                                                                         |
| OpenRouter Chat Model1       | @n8n/n8n-nodes-langchain.lmChatOpenRouter          | Secondary LLM for fallback                             | Fix Empty Response              | Set Output (If Empty)        |                                                                                                                                         |
| Score Text                   | @n8n/n8n-nodes-langchain.toolWorkflow               | Sub-workflow node to score text on performance & preference | AI Agent                    | AI Agent                    | Use the [Score Text sub-workflow](https://github.com/Open-Paws/Open-Paws-Documentation/blob/main/Automation/Scoring_Text_Sub_Workflow.json) |
| Sticky Note                  | n8n-nodes-base.stickyNote                            | Setup instructions and API key configuration guide   | -                                | -                            | Contains detailed setup and credential instructions                                                                                     |
| Sticky Note1                 | n8n-nodes-base.stickyNote                            | Workflow overview and architecture explanation        | -                                | -                            | Describes workflow purpose, integrated tools, and special sub-workflow for text scoring                                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Nodes:**

    - Add **When chat message received** node (Type: `@n8n/n8n-nodes-langchain.chatTrigger`) with default webhook properties.
    - Add **When Executed by Another Workflow** node (Type: `n8n-nodes-base.executeWorkflowTrigger`) configured to accept inputs: `chatInput` (string), `sessionId` (string).

2. **Add "Set Fields" Node:**

    - Type: `n8n-nodes-base.set`
    - Purpose: Map incoming JSON fields to variables `chatInput` and `sessionId`.
    - Parameters: Assign `chatInput = {{$json.chatInput}}`, `sessionId = {{$json.sessionId}}`.
    - Connect outputs of both trigger nodes to this node.

3. **Add "Simple Memory" Node:**

    - Type: `@n8n/n8n-nodes-langchain.memoryBufferWindow`
    - Parameters:  
      - `sessionKey`: expression `{{$('Set Fields').item.json.sessionId}}`  
      - `sessionIdType`: `customKey`  
      - `contextWindowLength`: `10`  
    - Purpose: Maintain conversation context for the session.

4. **Add "OpenRouter Chat Model" Node:**

    - Type: `@n8n/n8n-nodes-langchain.lmChatOpenRouter`
    - Parameters:  
      - Model: `google/gemini-2.5-flash-preview`  
      - Credentials: OpenRouter API (set up separately)  
    - Purpose: LLM API for the AI Agent.

5. **Add External Tool Integration Nodes:**

    - **Serper API:**  
      - Type: `@n8n/n8n-nodes-langchain.toolHttpRequest`  
      - URL: `https://google.serper.dev/{endpoint}` with placeholder for endpoint  
      - Auth: HTTP Header Auth with API key  
      - Use to perform live Google searches.

    - **Database Retrieval:**  
      - Type: `@n8n/n8n-nodes-langchain.toolHttpRequest`  
      - URL: Open Paws GraphQL endpoint  
      - Method: POST  
      - Body: GraphQL query with user query embedded  
      - Auth: HTTP Custom Auth with headers for Open Paws and OpenAI keys  

    - **Web Scraper Tool:**  
      - Type: `@n8n/n8n-nodes-langchain.toolHttpRequest`  
      - URL: `https://r.jina.ai/{url}`  
      - Auth: HTTP Header Auth with Jina AI API key

    - **Twitter Post Scraper:**  
      - Type: `@n8n/n8n-nodes-langchain.toolHttpRequest`  
      - URL: `http://api.scrapingdog.com/x/post?tweetId={tweetId}&parsed=true`  
      - Auth: HTTP Query Auth with Scrapingdog API key

    - **Twitter Profile Scraper:**  
      - Type: `@n8n/n8n-nodes-langchain.toolHttpRequest`  
      - URL: `http://api.scrapingdog.com/x/profile?profileId={profileId}&parsed=true`  
      - Auth: HTTP Query Auth with Scrapingdog API key

    - **Instagram Profile Scraper:**  
      - Type: `@n8n/n8n-nodes-langchain.toolHttpRequest`  
      - URL: `https://api.scrapingdog.com/instagram/profile?username={username}`  
      - Auth: HTTP Query Auth with Scrapingdog API key

    - **Linkedin Person and Company Scraper:**  
      - Type: `@n8n/n8n-nodes-langchain.toolHttpRequest`  
      - URL: `https://api.scrapingdog.com/linkedin?type=profile&linkId={linkedinID}&private=false`  
      - Auth: HTTP Query Auth with Scrapingdog API key

    - **Email Finder:**  
      - Type: `n8n-nodes-base.hunterTool`  
      - Parameters: domain, firstname, lastname mapped from AI variables  
      - Credentials: Hunter.io API

    - **Email Verifier:**  
      - Type: `n8n-nodes-base.hunterTool`  
      - Parameter: email from AI variables  
      - Credentials: Hunter.io API

6. **Add Core "AI Agent" Node:**

    - Type: `@n8n/n8n-nodes-langchain.agent`
    - Parameters:  
      - Text input: `={{ $('Set Fields').item.json.chatInput }}`  
      - Max iterations: `100`  
      - System message: Paste detailed prompt enforcing autonomous, multi-tool agent behavior  
      - Return intermediate steps: enabled  
    - Connect inputs:  
      - From `Set Fields` (text input)  
      - From `Simple Memory` (memory interface)  
      - From all external tool nodes (as tool inputs)  
      - From `OpenRouter Chat Model` (language model)  
    - Connect output to decision node.

7. **Add "If" Node to Check Agent Output:**

    - Type: `n8n-nodes-base.if`  
    - Condition: Check if `{{$json.output}}` is empty string

8. **Add Fallback Chain for Empty Output:**

    - **Fix Empty Response:**  
      - Type: `@n8n/n8n-nodes-langchain.chainLlm`  
      - Prompt: Combine user query and intermediate steps to regenerate answer  
      - Connect input from `If` node’s empty branch

    - **OpenRouter Chat Model1:**  
      - Same config as first OpenRouter Chat Model  
      - Connect input from Fix Empty Response node

    - **Set Output (If Empty):**  
      - Type: `n8n-nodes-base.set`  
      - Assign `output` and `intermediateSteps` fields  
      - Connect output of OpenRouter Chat Model1

9. **Add "Set Output (If Not Empty)" Node:**

    - Type: `n8n-nodes-base.set`  
    - Assign `output` and `intermediateSteps` from AI Agent output  
    - Connect output of `If` node’s non-empty branch

10. **Add "Score Text" Sub-Workflow Node:**

    - Type: `@n8n/n8n-nodes-langchain.toolWorkflow`  
    - Reference the external "Score Text" sub-workflow (upload separately)  
    - Input parameter: `text` mapped from AI or retrieved content  
    - Connect input and output appropriately to AI Agent (for enrichment or evaluation)

11. **Connect Final Outputs:**

    - Both `Set Output (If Empty)` and `Set Output (If Not Empty)` nodes produce the final workflow output containing the answer and intermediate steps.

12. **Credential Setup:**

    - Create and configure the following credentials in n8n:  
      - OpenRouter API (native or HTTP header)  
      - Serper API (HTTP header with X-API-KEY)  
      - Jina AI (HTTP header with Bearer token)  
      - Scrapingdog (HTTP query with API key)  
      - Hunter.io (native credential)  
      - Open Paws Database (HTTP custom with Authorization bearer and X-OpenAI-Api-Key headers)

13. **Testing and Validation:**

    - Test each external API node with sample inputs to confirm connectivity.  
    - Test the entire workflow with sample chat queries to verify autonomous multi-tool operation and final output generation.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      | Context or Link                                                                                                 |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------|
| **Setup instructions and credential configuration:** Upload the [Score Text sub-workflow](https://github.com/Open-Paws/Open-Paws-Documentation/blob/main/Automation/Scoring_Text_Sub_Workflow.json) to your n8n instance. Obtain API keys for OpenRouter, Serper, Scraping Dog, Jina AI, Hunter.io, and OpenAI. Configure credentials in n8n using Generic Credential Types or native types where available. The Open Paws DB retrieval requires a custom auth with two headers: Authorization (Bearer key) and X-OpenAI-Api-Key. Serper uses HTTP Header Auth with `X-API-KEY`. Jina AI uses HTTP Header Auth with Bearer token. Scraping Dog uses HTTP Query Auth with `api_key`. Hunter.io and OpenRouter have native credential types in n8n. | Credential setup instructions and links in the workflow’s sticky note.                                         |
| **Workflow overview:** This workflow acts as a foundational autonomous research agent for animal advocacy, integrating specialized APIs for domain-specific knowledge, live web search, social media scraping, and email discovery. It employs a multi-tool AI agent architecture with strict non-interactive autonomous execution guidelines, designed to produce complete, high-quality answers. Includes advanced error recovery and fallback mechanisms. The text scoring sub-workflow assists in evaluating content quality and advocacy alignment.                                                                                                                                                                                                                                                                                                                                                                                                                                                                 | See Sticky Note1 in workflow for detailed architecture and use case explanation.                              |
| **Open Paws Knowledge Base:** The Open Paws DB is a curated resource specialized in veganism, animal ethics, and advocacy topics. Queries run via GraphQL hybrid search using OpenAI embeddings enable precise, relevant information retrieval beyond generic search engines.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               | [Open Paws Documentation - Knowledge](https://github.com/Open-Paws/Open-Paws-Documentation/tree/main/Knowledge) |
| **Agent Operating Principles:** The AI agent must never ask users questions, always complete tasks fully, try all possible tool combinations, and provide a final comprehensive answer. It uses strategic reasoning ('Think' tool) before and after each step to optimize tool usage and solution completeness.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 | Embedded in AI Agent’s system message configuration.                                                          |
| **Error Recovery:** If a tool fails, the agent tries alternatives, reconsiders strategy, and never returns incomplete results or blank outputs.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          | Embedded in AI Agent system message and fallback nodes.                                                       |

---

**Disclaimer:** The provided text is extracted exclusively from an automated workflow created with n8n, adhering strictly to content policies. It contains no illegal or protected content. All processed data are legal and public.