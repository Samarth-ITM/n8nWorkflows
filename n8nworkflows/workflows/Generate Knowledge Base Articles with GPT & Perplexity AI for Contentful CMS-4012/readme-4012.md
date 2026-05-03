Generate Knowledge Base Articles with GPT & Perplexity AI for Contentful CMS

https://n8nworkflows.xyz/workflows/generate-knowledge-base-articles-with-gpt---perplexity-ai-for-contentful-cms-4012


# Generate Knowledge Base Articles with GPT & Perplexity AI for Contentful CMS

### 1. Workflow Overview

This workflow automates the generation and publication of structured knowledge base articles starting from a simple chat prompt. It is designed for content teams or automated systems that want to create, refine, and publish rich, well-researched articles into Contentful CMS with minimal manual intervention.

The workflow is logically divided into the following blocks:

**1.1 Input Reception**  
- Receives a user chat prompt via webhook to start the article generation.

**1.2 AI Drafting Loop**  
- Uses an AI Writer Agent to create an initial article draft in JSON format.

**1.3 Perplexity Research Call**  
- Enriches the article body with deep research content from Perplexity AI.

**1.4 Formatting and Citation Extraction**  
- Parses and formats the Perplexity AI output, appending citations.

**1.5 Editorial Loop**  
- Iteratively refines the article up to 3 times through an AI Editor Agent, merging improvements as needed to ensure quality.

**1.6 Contentful Publishing**  
- Publishes the finalized, reviewed article JSON into Contentful CMS.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
Receives user chat prompts as HTTP POST requests and initializes the workflow.

- **Nodes Involved:**  
  - When chat message received  
  - Initialize Count

- **Node Details:**  

**When chat message received**  
- Type: Chat Trigger (Webhook)  
- Role: Entry point that listens for chat prompts (e.g., "What topics should I write about?")  
- Configuration: Public webhook, default initial message set  
- Input: HTTP POST JSON payload with `chatInput`  
- Output: Passes chat input downstream  
- Failure Cases: Webhook downtime, unexpected payload formats

**Initialize Count**  
- Type: Set Node  
- Role: Initializes iteration count variable `iterationCount` to 0 for editorial loop control  
- Input: Receives chat input data  
- Output: Adds `iterationCount` to dataset  
- Failure Cases: None expected, but missing initialization may cause infinite loops

---

#### 2.2 AI Drafting Loop

- **Overview:**  
Generates the initial article draft or revises an existing one using the AI Writer Agent.

- **Nodes Involved:**  
  - AI Writer Agent  
  - OpenAI Chat Model  
  - Create Perplexity Content  
  - JSON.parse1  
  - Merge

- **Node Details:**  

**AI Writer Agent**  
- Type: LangChain AI Agent node  
- Role: Produces or updates article JSON including metadata and content  
- Configuration:  
  - Input: Either new chat prompt or existing article JSON with improvements  
  - Output format: Strict JSON with fields like title, slug, category.id, description, keywords array, content, metaTitle, metaDescription, readingTime, difficulty  
  - System message: Specifies structure and instructions to not alter title or category unless creating new article  
- Key Expressions: Uses input JSON fields (`chatInput`, `title`, `improvements`) for conditional drafting  
- Connected to: OpenAI Chat Model (gpt-3.5-turbo) for generation  
- Failure Cases: AI output invalid JSON, API key issues, timeouts

**OpenAI Chat Model**  
- Type: Language Model Chat (OpenAI GPT-3.5-turbo)  
- Role: Language engine for AI Writer Agent  
- Credentials: OpenAI API key configured  
- Failure Cases: API rate limits, invalid keys, network issues

**Create Perplexity Content**  
- Type: HTTP Request  
- Role: Calls Perplexity AI API to generate deeply researched content on the article title with improvements incorporated  
- Configuration:  
  - POST to `https://api.perplexity.ai/chat/completions`  
  - Body includes model "sonar-deep-research," user message with article title and improvements, max tokens 60000  
  - Headers contain Perplexity API bearer token  
- Output: Raw JSON with research content and citations  
- Failure Cases: API auth failures, quota limits, malformed request or response

**JSON.parse1**  
- Type: Code Node (JavaScript)  
- Role: Parses JSON string from Perplexity API response into structured JSON object for downstream use  
- Failure Cases: JSON parse errors if response malformed

**Merge**  
- Type: Merge Node  
- Role: Combines multiple inputs: article metadata, enriched content, iteration count for next steps  
- Failure Cases: Data mismatch or missing inputs causing merge errors

---

#### 2.3 Formatting and Citation Extraction

- **Overview:**  
Formats the Perplexity AI output content by cleaning and appending citation links as markdown.

- **Nodes Involved:**  
  - Format Perplexity Output & Add Citations (Code Node)

- **Node Details:**  

**Format Perplexity Output & Add Citations**  
- Type: Code Node  
- Role:  
  - Removes any `<think>` blocks in content to clean text  
  - Converts citations array into markdown bullet list with numbered links  
  - Appends citations section to article content  
- Failure Cases: Missing fields (`choices`, `citations`), unexpected data formats causing runtime errors

---

#### 2.4 Editorial Loop

- **Overview:**  
Refines the article draft by iterative evaluation and improvements using AI Editor Agent, with safeguards against infinite loops.

- **Nodes Involved:**  
  - AI Editor Agent  
  - OpenAI Chat Model1  
  - Check Limit (If Node)  
  - should submit (If Node)  
  - Increment Count  
  - Merge  
  - Stop Here  
  - JSON.parse  
  - JSON.parse3  
  - Format  
  - Accept and Publish (Tool Workflow)  
  - Execute Workflow (Publish to Contentful sub-workflow)

- **Node Details:**  

**AI Editor Agent**  
- Type: LangChain AI Agent node  
- Role: Reviews article JSON for clarity, quality, and completeness; decides whether to rewrite or submit  
- Configuration:  
  - Input: Article JSON with metadata and content  
  - Output: JSON with `action` field ("rewrite" or "submit"), `improvements` if rewriting, plus all original fields  
  - System message: Detailed prompt specifying evaluation criteria and strict output format  
- Connected to: OpenAI Chat Model1 (GPT-4.5-preview) for advanced editorial capabilities  
- Failure Cases: Invalid JSON output, API errors

**OpenAI Chat Model1**  
- Type: Language Model Chat (OpenAI GPT-4.5-preview)  
- Role: Language engine for AI Editor Agent  
- Credentials: Same OpenAI API account as writer agent  
- Failure Cases: API limits, connectivity issues

**Check Limit**  
- Type: If Node  
- Role: Determines if iteration count reached or exceeded 3 to stop further editing  
- Failure Cases: Incorrect count variable may cause premature or no stopping

**should submit**  
- Type: If Node  
- Role: Checks if AI Editor Agent’s output action is "submit" to proceed to publishing  
- Failure Cases: Improper parsing of action field

**Increment Count**  
- Type: Function Node  
- Role: Increments iteration count after each editorial cycle to control loop  
- Failure Cases: Missing or corrupted iteration count input

**Merge**  
- Role: Combines updated article JSON, enriched content, and iteration count for next loop or publishing

**Stop Here**  
- Type: NoOp Node  
- Role: Acts as loop exit point when iteration limit reached

**JSON.parse / JSON.parse3**  
- Type: Code Nodes for parsing JSON strings from AI outputs before further processing

**Format**  
- Type: Code Node  
- Role: Merges the updated content field and iteration count into the article JSON for next iteration

**Accept and Publish**  
- Type: Tool Workflow Node  
- Role: Triggers a sub-workflow to publish the final article to Contentful upon passing editorial checks  
- Configuration: Passes article JSON fields as inputs  
- Failure Cases: Sub-workflow failure, input mismatch

**Execute Workflow**  
- Type: Execute Workflow Node  
- Role: Invokes the external "Publish to Contentful" workflow with finalized article data  
- Failure Cases: Workflow ID invalid, parameter mismatch, execution errors

---

#### 2.5 Contentful Publishing

- **Overview:**  
Publishes the reviewed and finalized article JSON into Contentful CMS, converting fields to required formats and using Contentful’s API.

- **Nodes Involved:**  
  - (Sub-workflow triggered by Accept and Publish and Execute Workflow nodes; not detailed here but referenced)

- **Node Details:**  
- The publishing workflow maps JSON fields to Contentful entry fields  
- Uses HTTP PUT with authorization headers to update Contentful entries  
- Converts article content to Contentful Rich Text format using separate AI formatter (external workflow)  
- Credentials: Contentful API token must be configured  
- Failure Cases: Authentication errors, API rate limits, field mapping errors

---

### 3. Summary Table

| Node Name                        | Node Type                           | Functional Role                              | Input Node(s)                          | Output Node(s)                     | Sticky Note                                                                                         |
|---------------------------------|-----------------------------------|----------------------------------------------|--------------------------------------|----------------------------------|---------------------------------------------------------------------------------------------------|
| When chat message received       | Chat Trigger                      | Entry webhook receiving chat prompt          | —                                    | Initialize Count                  |                                                                                                   |
| Initialize Count                 | Set Node                         | Initializes editorial iteration count        | When chat message received            | AI Writer Agent, Increment Count |                                                                                                   |
| AI Writer Agent                 | LangChain AI Agent               | Generates initial article draft JSON         | Initialize Count                     | Create Perplexity Content         | Focuses on writing all fields; specified input/output format; implements editor feedback          |
| OpenAI Chat Model               | Language Model (OpenAI GPT-3.5)  | Provides language generation for Writer Agent| AI Writer Agent                      | AI Writer Agent                   |                                                                                                   |
| Create Perplexity Content       | HTTP Request                    | Requests detailed research content from Perplexity AI| AI Writer Agent                  | Format Perplexity Output & Add Citations |                                                                                                   |
| Format Perplexity Output & Add Citations | Code Node                   | Cleans Perplexity output, formats citations   | Create Perplexity Content            | Merge                            |                                                                                                   |
| Merge                          | Merge Node                      | Combines article metadata, content, iteration count | Format Perplexity Output & Add Citations, JSON.parse1, Increment Count | Format                         |                                                                                                   |
| Format                         | Code Node                      | Merges latest content and iteration count into article JSON | Merge                             | AI Editor Agent                   |                                                                                                   |
| AI Editor Agent                | LangChain AI Agent              | Reviews and suggests improvements or approval| Format                              | Check Limit                      | Reviews quality of output; decides publishability                                                 |
| OpenAI Chat Model1             | Language Model (OpenAI GPT-4.5) | Provides language generation for Editor Agent| AI Editor Agent                     | AI Editor Agent                  |                                                                                                   |
| Check Limit                   | If Node                        | Stops editorial loop after 3 iterations       | AI Editor Agent                    | Stop Here, should submit          | Tracks count to prevent infinite loop                                                             |
| should submit                 | If Node                        | Checks if article ready for submission         | Check Limit                       | JSON.parse3, JSON.parse           |                                                                                                   |
| Stop Here                    | NoOp Node                      | Loop exit point                                | Check Limit                       | JSON.parse3                      |                                                                                                   |
| Increment Count              | Function Node                  | Increments editorial iteration count           | AI Writer Agent, Merge             | Merge                           | Tracks a variable count to avoid infinite feedback loops                                          |
| JSON.parse1                  | Code Node                     | Parses JSON string from Perplexity response    | Create Perplexity Content          | Merge                           |                                                                                                   |
| JSON.parse                   | Code Node                     | Parses JSON string output from AI Editor Agent | should submit, AI Editor Agent     | AI Writer Agent, Increment Count |                                                                                                   |
| JSON.parse3                  | Code Node                     | Parses JSON string for final publishing step    | should submit, Stop Here           | Execute Workflow                 |                                                                                                   |
| Accept and Publish           | Tool Workflow                 | Triggers sub-workflow to publish article to Contentful | AI Editor Agent                 | AI Editor Agent                 |                                                                                                   |
| Execute Workflow             | Execute Workflow             | Calls external workflow to publish article      | JSON.parse3                      | —                               | Publishes article to Contentful CMS                                                              |
| Sticky Note                  | Sticky Note                   | Commentary on Writer Agent                      | —                                | —                               | Focuses on writing for all Contentful fields, specified format, handles editor feedback          |
| Sticky Note1                 | Sticky Note                   | Commentary on Count Incrementer                  | —                                | —                               | Tracks variable count to prevent infinite feedback loops                                         |
| Sticky Note2                 | Sticky Note                   | Commentary on Editor Agent                        | —                                | —                               | Sole purpose: evaluate quality and decide publishability                                        |
| Sticky Note3                 | Sticky Note                   | Commentary on Publish to Contentful               | —                                | —                               | Publishes to Contentful, converts article to Rich Text; contact christian@varritech.com for details |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Trigger:**  
   - Add a "Chat Trigger" node.  
   - Configure webhook for POST requests with path `/webhook/knowledge-article`.  
   - Make public for testing or secure as needed.  
   - Initial message example: "What topics should I write about?"  
   - Outputs JSON with field `chatInput`.

2. **Initialize Editorial Loop Counter:**  
   - Add a "Set" node named "Initialize Count".  
   - Set a new number variable `iterationCount` initialized to 0.  
   - Connect from the webhook trigger.

3. **Add AI Writer Agent:**  
   - Add a LangChain AI Agent node named "AI Writer Agent".  
   - Configure input text with prompt to create or revise article based on `chatInput` or existing article JSON.  
   - Enforce output JSON format with fields: title, slug, category.id, description, keywords[], content, metaTitle, metaDescription, readingTime, difficulty.  
   - Use a system message that instructs to keep the title fixed, pick category from specified list if new, and apply improvements if given.  
   - Connect "Initialize Count" node to this agent.

4. **Link AI Writer Agent to OpenAI Chat Model:**  
   - Add an "OpenAI Chat Model" node.  
   - Choose model `gpt-3.5-turbo`.  
   - Connect to "AI Writer Agent" as its language model.  
   - Configure OpenAI API credentials.

5. **Add HTTP Request to Perplexity AI:**  
   - Add an HTTP Request node named "Create Perplexity Content".  
   - POST to `https://api.perplexity.ai/chat/completions`.  
   - JSON body: includes model "sonar-deep-research", user message with article title and improvements, max tokens 60000, temperature 0.7, etc.  
   - Set Authorization header with Perplexity API bearer token.  
   - Connect output of "AI Writer Agent" to this node.

6. **Parse Perplexity Response JSON:**  
   - Add a Code node "JSON.parse1".  
   - JavaScript code to parse the raw JSON string from `output` field.  
   - Connect from "Create Perplexity Content".

7. **Format Perplexity Output and Add Citations:**  
   - Add a Code node "Format Perplexity Output & Add Citations".  
   - JS code to clean `<think>` blocks from content, convert citations array to markdown list, and append to content.  
   - Connect from "Create Perplexity Content".

8. **Merge Article Parts and Iteration Count:**  
   - Add a Merge node with 3 inputs.  
   - Connect inputs:  
     - From "Format Perplexity Output & Add Citations"  
     - From "JSON.parse1"  
     - From "Increment Count" (to be created later)  
   - Output combined article JSON for next step.

9. **Format Merged Data:**  
   - Add a Code node "Format".  
   - JS code merges updated content and iterationCount into first item JSON.  
   - Connect from "Merge".

10. **Add AI Editor Agent:**  
    - Add LangChain AI Agent node "AI Editor Agent".  
    - Input: article JSON with all metadata and content fields.  
    - Prompt: instruct to review article quality, suggest improvements or submit, return JSON with action and improvements.  
    - Connect "Format" node output to this node.

11. **Link AI Editor Agent to OpenAI Chat Model (GPT-4.5-preview):**  
    - Add OpenAI Chat Model node with model `gpt-4.5-preview`.  
    - Connect to "AI Editor Agent".  
    - Use same OpenAI credentials.

12. **Check Iteration Limit:**  
    - Add an If node "Check Limit".  
    - Condition: `iterationCount >= 3`.  
    - Connect output of "AI Editor Agent" to this node.

13. **Conditional Branching:**  
    - If limit reached, connect to "Stop Here" node (NoOp).  
    - If not, add If node "should submit" to check if `action == "submit"`.  
    - If submit, proceed to publishing nodes.  
    - If rewrite, continue editorial loop.

14. **Increment Iteration Count:**  
    - Add a Function node "Increment Count".  
    - JS code: increment `iterationCount` by 1.  
    - Connect from AI Writer Agent and from editorial loop for iterations.

15. **Use Merge Node to combine updated article and iteration count for next loop:**  
    - Connect "Increment Count" to "Merge" as the 3rd input.

16. **Parse JSON responses after editorial agent:**  
    - Add Code nodes "JSON.parse" and "JSON.parse3" to parse AI Editor Agent JSON outputs before further processing.

17. **Format merged article JSON for next iteration:**  
    - Connect parsed outputs back to "AI Writer Agent" or to publishing workflow accordingly.

18. **Add Accept and Publish tool workflow node:**  
    - Configure to call external publishing workflow by workflow ID.  
    - Map article JSON fields as inputs.  
    - Connect from editorial node on submit condition.

19. **Add Execute Workflow node:**  
    - Calls "Publish to Contentful" workflow.  
    - Pass finalized article JSON fields as parameters.

20. **Set up the Publish to Contentful sub-workflow (external):**  
    - Configure HTTP PUT request to Contentful API endpoint with entry ID and space.  
    - Use Contentful API token in headers.  
    - Map article JSON fields to Contentful entry schema.  
    - Handle Rich Text content conversion with AI formatter if desired (optional external flow).

21. **Add Sticky Notes for documentation:**  
    - Add notes to document Writer Agent, Editorial Loop count increment, Editor Agent role, and Publish to Contentful specifics.

22. **Configure error handling and timeouts:**  
    - Set workflow error workflow as needed.  
    - Set execution timeout (e.g., 1800s).

---

### 5. General Notes & Resources

| Note Content                                                                                          | Context or Link                                                                                         |
|-----------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| Writer Agent focuses on writing all Contentful fields and implementing editor feedback               | Sticky Note near "AI Writer Agent" node                                                                |
| Editorial Loop counter prevents infinite refinement loops                                           | Sticky Note near "Increment Count" and "Check Limit" nodes                                             |
| Editor Agent solely evaluates content quality and decides publishability                             | Sticky Note near "AI Editor Agent" node                                                                |
| Publish to Contentful converts article to Rich Text format using another AI formatter; contact christian@varritech.com for details | Sticky Note near publishing nodes, external formatter workflow not included in this workflow JSON      |

---

This document provides a complete reference for the "Auto Knowledge Base Article Generator" n8n workflow, enabling developers and automation agents to understand, reproduce, and maintain it effectively.