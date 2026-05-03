SEO Blog Generator with GPT-4o, Perplexity, and Telegram Integration

https://n8nworkflows.xyz/workflows/seo-blog-generator-with-gpt-4o--perplexity--and-telegram-integration-3672


# SEO Blog Generator with GPT-4o, Perplexity, and Telegram Integration

### 1. Workflow Overview

This workflow automates the generation of SEO-optimized blog posts by leveraging AI technologies and integrates optional Telegram interaction for triggering and delivering content. It is designed for marketing teams or content creators aiming to produce authoritative, well-structured blog posts with relevant metadata for recruitment or HR-related topics, but adaptable to other domains.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception**: Captures user input either via a web form or Telegram messages to initiate the blog generation process.
- **1.2 Research Gathering (Perplexity Searcher Sub-Workflow)**: Uses a sub-workflow powered by Perplexity.ai to fetch up-to-date research data based on the input query.
- **1.3 AI Content Generation**: Employs multiple AI agents and OpenAI GPT-4o models to generate blog content, extract structured metadata, and create SEO-friendly titles, slugs, and meta descriptions.
- **1.4 Output Assembly and Delivery**: Combines generated content and metadata, optionally sends the final blog post to Telegram, and aggregates output data for further use.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block captures the initial user input that triggers the workflow. It supports two input methods: a web form submission and Telegram messages.

- **Nodes Involved:**  
  - On form submission (Form Trigger)  
  - Tele HoangSP_Social_Media (Telegram Trigger)  
  - AI Agent (initial processing of input text)

- **Node Details:**

  - **On form submission**  
    - Type: Form Trigger  
    - Role: Listens for form submissions with a single required textarea field labeled "Research Query" where users enter the blog topic or query.  
    - Configuration: Form titled "Blog Factory" with description "Create SEO optimized blog posts".  
    - Inputs: User-submitted form data.  
    - Outputs: Passes the query to the AI Agent node.  
    - Edge Cases: Missing or empty input will prevent triggering; no fallback handling shown.

  - **Tele HoangSP_Social_Media**  
    - Type: Telegram Trigger  
    - Role: Listens for incoming Telegram messages to trigger the workflow.  
    - Configuration: Monitors "message" updates only.  
    - Inputs: Telegram message text.  
    - Outputs: Passes message text to the AI Agent node.  
    - Edge Cases: Requires proper Telegram Bot Token and webhook setup; message format must be text.

  - **AI Agent**  
    - Type: LangChain Agent  
    - Role: Processes the input query or Telegram message to prepare for research.  
    - Configuration: System message sets professional tone as a medical doctor specializing in Men's Health; prompt extracts the research query from either form or Telegram input.  
    - Key Expression: Uses `{{ $json['Research Query'] }}` or `{{ $json.message.text }}` to dynamically extract input.  
    - Inputs: Triggered by either form or Telegram node.  
    - Outputs: Passes query to Perplexity_Searcher sub-workflow.  
    - Edge Cases: Expression failures if input fields are missing; requires valid input text.

---

#### 2.2 Research Gathering (Perplexity Searcher Sub-Workflow)

- **Overview:**  
  This block invokes a sub-workflow named `Perplexity_Searcher` that queries Perplexity.ai API to retrieve the latest and relevant research data based on the input query.

- **Nodes Involved:**  
  - Perplexity_Searcher (Sub-Workflow Node)

- **Node Details:**

  - **Perplexity_Searcher**  
    - Type: LangChain Tool Workflow (Sub-Workflow)  
    - Role: Executes an external workflow designed to perform AI-powered research using Perplexity.ai.  
    - Configuration: References workflow ID `5uapJIjLLhwnhX0n` with a description indicating its purpose to fetch the latest information.  
    - Inputs: Receives the research query from the AI Agent node.  
    - Outputs: Returns detailed research findings for blog content generation.  
    - Edge Cases: Requires valid Perplexity API credentials; network or API errors may cause failure; sub-workflow must be imported and active.

---

#### 2.3 AI Content Generation

- **Overview:**  
  This block uses OpenAI GPT-4o and LangChain agents to generate the blog post content, extract structured metadata, and create SEO-optimized titles, slugs, and meta descriptions.

- **Nodes Involved:**  
  - Blog Content Generator (LangChain Agent)  
  - OpenAI Chat Model (GPT-4o-mini)  
  - Copywriter AI Agent (LangChain Agent)  
  - AI Agent1 (LangChain Agent)  
  - OpenAI Chat Model1 (GPT-4o-mini)  
  - OpenAI Chat Model2 (GPT-4o-mini)  
  - Structured Output Parser  
  - Metadata Generator (LangChain Agent)  
  - Structured Output Parser1  
  - Metadata Extractor (Structured Output Parser)  
  - Simple Memory, Simple Memory1 (Memory Buffer Nodes)

- **Node Details:**

  - **Blog Content Generator**  
    - Type: LangChain Agent  
    - Role: Generates a detailed, SEO-optimized blog post draft based on the research findings and query.  
    - Configuration: Prompt instructs writing a 1500-2000 word blog post for Men's Health Consulting industry with specific SEO and content requirements, including keyword integration and CTA.  
    - Input: Receives research output from Perplexity_Searcher.  
    - Output: Blog post text passed to Copywriter AI Agent.  
    - Edge Cases: Large output size may cause timeouts; prompt clarity critical for quality.

  - **OpenAI Chat Model (multiple instances)**  
    - Type: GPT-4o-mini language model nodes  
    - Role: Provide language model processing for AI agents.  
    - Configuration: Model set to "gpt-4o-mini" with OpenAI API credentials.  
    - Inputs/Outputs: Linked to respective LangChain agents for content generation and metadata creation.  
    - Edge Cases: API quota limits, authentication errors, or network issues.

  - **Copywriter AI Agent**  
    - Type: LangChain Agent  
    - Role: Further processes blog content to refine or expand as needed.  
    - Inputs: Receives output from Blog Content Generator.  
    - Outputs: Feeds into Metadata Generator and AI Agent1.  
    - Edge Cases: Prompt or input inconsistencies may reduce output quality.

  - **Metadata Generator**  
    - Type: LangChain Agent with Output Parser  
    - Role: Creates SEO metadata (slug, title, meta description) from the blog post content.  
    - Configuration: Detailed prompt with strict guidelines for slug, title, and meta description creation focused on recruitment/HR keywords and SEO best practices.  
    - Input: Receives blog post content from Copywriter AI Agent.  
    - Output: JSON object with slug, title, and meta description parsed by Structured Output Parser.  
    - Edge Cases: Parsing failures if output is malformed; prompt adherence critical.

  - **AI Agent1**  
    - Type: LangChain Agent with Output Parser  
    - Role: Extracts structured JSON content including title, subtitle, content, and hashtags from blog text.  
    - Configuration: Prompt instructs extraction of JSON object with specified fields; output parsed by Structured Output Parser1.  
    - Input: Receives refined blog content from Copywriter AI Agent.  
    - Output: Parsed JSON passed to Merge node.  
    - Edge Cases: Missing fields inferred; malformed JSON may cause parsing errors.

  - **Structured Output Parser & Structured Output Parser1**  
    - Type: LangChain Output Parsers  
    - Role: Parse AI-generated text into structured JSON objects according to defined schemas.  
    - Configuration: JSON schema examples provided for validation.  
    - Inputs: Receive raw AI output.  
    - Outputs: Validated JSON objects for downstream processing.  
    - Edge Cases: Parsing failures if AI output deviates from schema.

  - **Metadata Extractor**  
    - Type: Structured Output Parser  
    - Role: Parses detailed metadata including title, subtitle, content, and hashtags from AI Agent1 output.  
    - Configuration: Manual JSON schema specifying required fields.  
    - Edge Cases: Missing required fields cause errors.

  - **Simple Memory & Simple Memory1**  
    - Type: LangChain Memory Buffer Window  
    - Role: Maintain context windows for AI agents to improve coherence in multi-turn interactions.  
    - Configuration: Session key dynamically set from AI output; context window length 10.  
    - Inputs/Outputs: Connected to AI Agent1 and Metadata Generator respectively.  
    - Edge Cases: Session key must be unique and consistent; memory overflow if context too large.

---

#### 2.4 Output Assembly and Delivery

- **Overview:**  
  This block merges the generated blog content and metadata, aggregates all details, and optionally sends the final blog post to a Telegram chat.

- **Nodes Involved:**  
  - Merge  
  - Combine Blog Details (Aggregate)  
  - Telegram

- **Node Details:**

  - **Merge**  
    - Type: Merge Node  
    - Role: Combines outputs from Metadata Generator and AI Agent1 to unify metadata and blog content.  
    - Configuration: Mode set to "combine" by position.  
    - Inputs: Receives JSON objects from Metadata Generator and AI Agent1.  
    - Outputs: Passes combined data to Combine Blog Details and Telegram nodes.  
    - Edge Cases: Mismatched input lengths may cause incomplete merges.

  - **Combine Blog Details**  
    - Type: Aggregate Node  
    - Role: Aggregates all item data into a single output object for final use or export.  
    - Configuration: Aggregates all item data without additional options.  
    - Inputs: Receives merged data from Merge node.  
    - Outputs: Final combined blog post data.  
    - Edge Cases: Large data may impact performance.

  - **Telegram**  
    - Type: Telegram Node (Send Message)  
    - Role: Sends the final blog post content and metadata to the Telegram chat that triggered the workflow.  
    - Configuration: Message text composed of title, subtitle, and content fields from merged output; chat ID dynamically extracted from Telegram trigger node.  
    - Inputs: Receives merged blog post data.  
    - Credentials: Requires Telegram Bot Token with proper permissions.  
    - Edge Cases: Invalid chat ID or Telegram API errors; message length limits.

---

### 3. Summary Table

| Node Name               | Node Type                              | Functional Role                             | Input Node(s)                  | Output Node(s)                 | Sticky Note                                                                                 |
|-------------------------|--------------------------------------|---------------------------------------------|-------------------------------|-------------------------------|---------------------------------------------------------------------------------------------|
| On form submission       | Form Trigger                         | Receives user blog topic input via form    | —                             | AI Agent                      | ## Perplexity Research                                                                     |
| Tele HoangSP_Social_Media| Telegram Trigger                    | Receives user blog topic input via Telegram| —                             | AI Agent                      | ## Perplexity Research                                                                     |
| AI Agent                | LangChain Agent                      | Processes input query for research          | On form submission, Telegram  | Perplexity_Searcher            | ## Perplexity Research                                                                     |
| Perplexity_Searcher      | LangChain Tool Workflow (Sub-Workflow)| Fetches latest research data from Perplexity| AI Agent                      | Blog Content Generator         |                                                                                             |
| Blog Content Generator   | LangChain Agent                     | Generates SEO-optimized blog post draft     | Perplexity_Searcher            | Copywriter AI Agent            | ## Write SEO Optimized Blog Post                                                           |
| Copywriter AI Agent      | LangChain Agent                     | Refines blog content                         | Blog Content Generator         | Metadata Generator, AI Agent1  | ## Write SEO Optimized Blog Post                                                           |
| Metadata Generator       | LangChain Agent with Output Parser  | Creates SEO metadata (slug, title, meta)    | Copywriter AI Agent            | Structured Output Parser       | ## Create Title, Slug & Meta                                                                |
| Structured Output Parser | LangChain Output Parser             | Parses metadata JSON                         | Metadata Generator             | Create Title, Slug, Meta       | ## Create Title, Slug & Meta                                                                |
| AI Agent1               | LangChain Agent with Output Parser  | Extracts structured blog content JSON       | Copywriter AI Agent            | Structured Output Parser1      |                                                                                             |
| Structured Output Parser1| LangChain Output Parser             | Parses blog content JSON                      | AI Agent1                     | Merge                         |                                                                                             |
| Merge                   | Merge Node                         | Combines metadata and blog content           | Structured Output Parser, Structured Output Parser1 | Telegram, Combine Blog Details |                                                                                             |
| Combine Blog Details     | Aggregate Node                    | Aggregates all blog data into one object     | Merge                         | —                             |                                                                                             |
| Telegram                | Telegram Node                      | Sends final blog post to Telegram chat       | Merge                         | —                             |                                                                                             |
| OpenAI Chat Model        | GPT-4o-mini Language Model         | Provides GPT-4o model for Metadata Generator | Metadata Generator            | Metadata Generator            |                                                                                             |
| OpenAI Chat Model1       | GPT-4o-mini Language Model         | Provides GPT-4o model for AI Agent           | AI Agent                      | AI Agent                     |                                                                                             |
| OpenAI Chat Model2       | GPT-4o-mini Language Model         | Provides GPT-4o model for AI Agent1          | AI Agent1                    | AI Agent1                    |                                                                                             |
| Simple Memory           | Memory Buffer Window                | Maintains context for AI Agent1               | AI Agent1                    | AI Agent1                    |                                                                                             |
| Simple Memory1          | Memory Buffer Window                | Maintains context for Metadata Generator      | Metadata Generator           | Metadata Generator           |                                                                                             |
| Note4                   | Sticky Note                        | Notes on SEO blog post writing                | —                             | —                             | ## Write SEO Optimized Blog Post                                                           |
| Note5                   | Sticky Note                        | Notes on Perplexity research                   | —                             | —                             | ## Perplexity Research                                                                     |
| Note7                   | Sticky Note                        | Notes on creating title, slug, and meta       | —                             | —                             | ## Create Title, Slug & Meta                                                                |
| Note                    | Sticky Note                        | Sample content and instructions for blog writing | —                             | —                             |                                                                                             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Input Nodes:**

   - Add a **Form Trigger** node named "On form submission":
     - Configure a form titled "Blog Factory".
     - Add a required textarea field labeled "Research Query" with a placeholder.
     - This node will trigger the workflow on form submission.

   - Add a **Telegram Trigger** node named "Tele HoangSP_Social_Media":
     - Set to listen for "message" updates.
     - Configure Telegram credentials with your Bot Token and webhook.

2. **Process Input Query:**

   - Add a **LangChain Agent** node named "AI Agent":
     - Set prompt to extract the research query from either form input (`{{ $json['Research Query'] }}`) or Telegram message (`{{ $json.message.text }}`).
     - Use a professional system message to set context (e.g., medical doctor specializing in Men's Health).
     - Connect both "On form submission" and "Tele HoangSP_Social_Media" nodes to this node.

3. **Invoke Research Sub-Workflow:**

   - Add a **LangChain Tool Workflow** node named "Perplexity_Searcher":
     - Reference the sub-workflow ID responsible for querying Perplexity.ai.
     - Ensure Perplexity API credentials are set.
     - Connect "AI Agent" output to this node.

4. **Generate Blog Content:**

   - Add a **LangChain Agent** node named "Blog Content Generator":
     - Configure a detailed prompt instructing to write a 1500-2000 word SEO-optimized blog post for Men's Health Consulting.
     - Input is the output from "Perplexity_Searcher".
     - Connect "Perplexity_Searcher" output to this node.

5. **Refine Blog Content:**

   - Add a **LangChain Agent** node named "Copywriter AI Agent":
     - Connect "Blog Content Generator" output to this node.
     - This node further processes or refines the blog content.

6. **Generate Metadata:**

   - Add a **LangChain Agent** node named "Metadata Generator":
     - Configure prompt to create slug, title, and meta description with detailed SEO guidelines focused on recruitment/HR keywords.
     - Connect "Copywriter AI Agent" output to this node.

   - Add an **OpenAI Chat Model** node (GPT-4o-mini) with OpenAI credentials:
     - Connect it as the language model for "Metadata Generator".

   - Add a **Structured Output Parser** node:
     - Configure with a JSON schema example for slug, title, and meta description.
     - Connect "Metadata Generator" output to this parser.

7. **Extract Structured Blog Content:**

   - Add a **LangChain Agent** node named "AI Agent1":
     - Configure prompt to extract JSON with fields: title, subtitle, content, hashtags.
     - Connect "Copywriter AI Agent" output to this node.

   - Add an **OpenAI Chat Model** node (GPT-4o-mini) with OpenAI credentials:
     - Connect as language model for "AI Agent1".

   - Add a **Structured Output Parser** node:
     - Configure to parse the JSON output from "AI Agent1".
     - Connect "AI Agent1" output to this parser.

   - Add a **Metadata Extractor** node (Structured Output Parser):
     - Configure manual JSON schema for title, subtitle, content, hashtags.
     - Connect "OpenAI Chat Model2" output to this node.

8. **Add Memory Buffers:**

   - Add two **Memory Buffer Window** nodes named "Simple Memory" and "Simple Memory1":
     - Configure session keys dynamically from AI outputs.
     - Connect "AI Agent1" and "Metadata Generator" respectively to these memory nodes to maintain context.

9. **Merge and Aggregate Outputs:**

   - Add a **Merge** node:
     - Set mode to "combine" by position.
     - Connect outputs from "Structured Output Parser" (metadata) and "Structured Output Parser1" (blog content).
   
   - Add an **Aggregate** node named "Combine Blog Details":
     - Configure to aggregate all item data.
     - Connect "Merge" output to this node.

10. **Send Output to Telegram:**

    - Add a **Telegram** node:
      - Configure to send a message to the chat ID extracted dynamically from the Telegram trigger node.
      - Message text includes title, subtitle, and content fields from merged output.
      - Connect "Merge" output to this node.
      - Use Telegram API credentials.

11. **Credential Setup:**

    - Add OpenAI API credentials with access to GPT-4o or GPT-3.5.
    - Add Perplexity API credentials with access to `/chat/completions`.
    - Add Telegram Bot credentials and configure webhook for Telegram trigger.

12. **Activate and Test:**

    - Activate the workflow.
    - Test by submitting a form or sending a Telegram message with a research query.
    - Verify that the blog post and metadata are generated and optionally sent to Telegram.

---

### 5. General Notes & Resources

| Note Content                                                                                           | Context or Link                                                                                   |
|------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| This workflow integrates Perplexity.ai for up-to-date research and OpenAI GPT-4o for content creation.| Workflow description and setup instructions.                                                    |
| Optional Telegram integration allows triggering and receiving blog posts via chat.                    | Requires Telegram Bot Token and webhook setup.                                                  |
| Prompt texts inside "Blog Content Generator" and "Metadata Generator" nodes can be customized to target different industries or writing styles. | Customization instructions in workflow description.                                            |
| For help or community support, visit the n8n community forum: https://community.n8n.io                | Official n8n community forum.                                                                   |
| Sticky notes provide contextual guidance on SEO blog writing, Perplexity research, and metadata creation. | Visible in workflow editor for user reference.                                                 |

---

This documentation provides a detailed, structured understanding of the SEO Blog Generator workflow, enabling reproduction, modification, and troubleshooting by advanced users and AI agents alike.