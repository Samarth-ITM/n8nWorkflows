Research-to-Social Post Generator with Perplexity AI and Facebook Graph API

https://n8nworkflows.xyz/workflows/research-to-social-post-generator-with-perplexity-ai-and-facebook-graph-api-8664


# Research-to-Social Post Generator with Perplexity AI and Facebook Graph API

### 1. Workflow Overview

This n8n workflow, titled **Research-to-Social Post Generator with Perplexity AI and Facebook Graph API**, automates the creation of engaging social media posts based on user chat input. It targets marketers, founders, and content teams who want to quickly convert conversational prompts into well-researched, polished, and localized social posts, specifically for Facebook, with optional publishing.

The workflow’s logic is organized into these main blocks:  

- **1.1 Chat Input Reception**  
  Captures user prompts from a chat interface.

- **1.2 Topic Analysis and Research**  
  Uses AI agents to analyze topics and optionally call an external research sub-workflow (Perplexity Researcher) to gather up-to-date information.

- **1.3 Structured Article Generation**  
  Parses AI outputs into a validated structured JSON article format with title, subtitle, content sections, and hashtags.

- **1.4 Post Content Creation**  
  Converts the structured article JSON into a concise, engaging Vietnamese social media post with emojis and calls to action.

- **1.5 Publishing to Facebook**  
  Posts the final content to a Facebook Page via the Graph API.

Each block is supported by configuration nodes, language model nodes (OpenAI GPT variants), output parsers, and sticky notes for documentation and guidance.

---

### 2. Block-by-Block Analysis

#### 2.1 Chat Input Reception

- **Overview:**  
  Listens for user chat input and initiates the workflow with a friendly greeting and prompt for content assistance.

- **Nodes Involved:**  
  - Chat Trigger  
  - Sticky Note: Chat Trigger

- **Node Details:**

  - **Chat Trigger**  
    - Type: Chat trigger node from Langchain integration  
    - Config: Public access; initial greeting message sets a conversational tone ("Hi there!...") inviting users to request research or post drafting.  
    - Inputs: External chat messages  
    - Outputs: Passes user input to the next agent node  
    - Edge Cases: Connection or permission issues with chat interface; malformed user input  
    - Notes: Sticky note advises to keep prompt simple, possibly with quick-help examples.

---

#### 2.2 Topic Analysis and Research

- **Overview:**  
  An AI agent analyzes the user’s topic, potentially calling a research sub-workflow (Perplexity Researcher) to retrieve current, high-quality information to support content creation.

- **Nodes Involved:**  
  - Agent: Topic + Research  
  - Tool: Call Perplexity Researcher  
  - LM: For Topic Agent (OpenAI GPT-4o-mini)  
  - Sticky Notes: Agent: Topic + Research, Tool: Call Perplexity Researcher, LM: For Topic Agent

- **Node Details:**

  - **Agent: Topic + Research**  
    - Type: Langchain AI agent node  
    - Config: Uses current timestamp and user chat input as prompt; system message instructs to use Perplexity Research tool if needed.  
    - Inputs: Chat Trigger output  
    - Outputs: Drafts a topic brief with research insights  
    - Edge Cases: Model API failures, improper prompt formatting, research tool unavailability  
    - Notes: Sticky note recommends concise instructions to agent.

  - **Tool: Call Perplexity Researcher**  
    - Type: Langchain tool workflow node  
    - Config: Calls an external research workflow (to be linked by setting `workflowId`) that performs Perplexity AI research.  
    - Inputs: Triggered by AI agent when research is required  
    - Outputs: Research results as a list of sources and takeaways  
    - Edge Cases: Missing or incorrect `workflowId`, sub-workflow errors, timeouts  
    - Notes: Sticky note stresses setting `workflowId` before running.

  - **LM: For Topic Agent**  
    - Type: Langchain language model node (OpenAI GPT-4o-mini)  
    - Config: Used by Topic + Research agent; credentials configured outside the node.  
    - Edge Cases: Authentication errors, API rate limits  
    - Notes: Sticky note highlights credential management.

---

#### 2.3 Structured Article Generation

- **Overview:**  
  Extracts and validates a structured JSON object representing the article (title, subtitle, main content, sections, quotes, hashtags) from the AI agent’s output.

- **Nodes Involved:**  
  - Agent: Extract JSON  
  - Parser: Article JSON  
  - LM: For JSON Extraction (OpenAI GPT-4o)  
  - Sticky Notes: Agent: Extract JSON, Parser: Article JSON, LM: For JSON Extraction

- **Node Details:**

  - **Agent: Extract JSON**  
    - Type: Langchain AI agent node  
    - Config: Prompt instructs to extract a JSON object from previous AI output, filling missing fields except quotes.  
    - Inputs: Output from Topic + Research agent  
    - Outputs: JSON object of article content  
    - Edge Cases: Parsing errors due to malformed text, incomplete extraction  
    - Notes: Sticky note advises declarative instructions.

  - **Parser: Article JSON**  
    - Type: Langchain structured output parser node  
    - Config: Manual JSON schema defining required article fields (title, subtitle, content with sections and quotes, hashtags).  
    - Inputs: Text output from Extract JSON agent  
    - Outputs: Validated structured JSON article  
    - Edge Cases: Schema mismatches, missing required fields  
    - Notes: Sticky note emphasizes upstream agent responsibility for completeness.

  - **LM: For JSON Extraction**  
    - Type: Langchain language model node (OpenAI GPT-4o)  
    - Config: Used by Extract JSON agent; temperature and other options configurable outside the node.  
    - Edge Cases: API errors, inconsistent JSON generation  
    - Notes: Sticky note suggests tuning parameters if needed.

---

#### 2.4 Post Content Creation

- **Overview:**  
  Converts the structured article JSON into a concise, engaging Vietnamese social media post, formatted with emojis, calls to action, and hashtags.

- **Nodes Involved:**  
  - Agent: Create Post Content  
  - LM: For Post Writer (OpenAI GPT-4o-mini)  
  - Parser: Post Fields  
  - Sticky Notes: Agent: Create Post Content, LM: For Post Writer, Parser: Post Fields

- **Node Details:**

  - **Agent: Create Post Content**  
    - Type: Langchain AI agent node  
    - Config: Prompt instructs conversion of JSON article to Vietnamese blog post with formatting guidelines (emojis, CTAs, easy readability, correct spelling).  
    - Inputs: JSON output from Extract JSON agent  
    - Outputs: Formatted social post content  
    - Edge Cases: Language model misinterpretation, formatting errors  
    - Notes: Sticky note highlights tone and style adjustments here.

  - **LM: For Post Writer**  
    - Type: Langchain language model node (OpenAI GPT-4o-mini)  
    - Config: Used by Create Post Content agent; lower temperature for consistency.  
    - Edge Cases: API failures, rate limits  
    - Notes: Sticky note mentions credential setup.

  - **Parser: Post Fields**  
    - Type: Langchain structured output parser node  
    - Config: Defines the final social post JSON fields: title, subtitle, content.  
    - Inputs: Agent output from Create Post Content  
    - Outputs: Structured social post JSON  
    - Edge Cases: Missing fields, parsing errors  
    - Notes: Sticky note explains downstream formatting role.

---

#### 2.5 Publishing to Facebook

- **Overview:**  
  Optionally publishes the final social post content to a Facebook Page using the Facebook Graph API.

- **Nodes Involved:**  
  - Publish: Facebook Graph API  
  - ⚙️ CONFIG (Set fields)  
  - Sticky Notes: Publish: Facebook Graph API, CONFIG

- **Node Details:**

  - **Publish: Facebook Graph API**  
    - Type: n8n native Facebook Graph API node  
    - Config: Posts to the `feed` edge of a Facebook Page specified by `YOUR_FACEBOOK_PAGE_ID`. Sends `message` parameter with post content (all asterisks removed).  
    - Inputs: Final social post content from Post Content Creation agent  
    - Outputs: Facebook API response  
    - Edge Cases: Auth errors, invalid Page ID, permissions issues, rate limits  
    - Notes: Sticky note recommends disabling during tests or adding review step.

  - **⚙️ CONFIG (Set fields)**  
    - Type: Set node  
    - Config: Holds configurable variables—Facebook Page ID, edge type, Research workflow ID, emoji style, language code. No credentials stored inside nodes.  
    - Inputs: None (start of flow)  
    - Outputs: Supplies constants to other nodes via expressions  
    - Edge Cases: Misconfigured or missing values cause downstream failures  
    - Notes: Sticky note encourages no hardcoded sensitive data.

---

### 3. Summary Table

| Node Name                      | Node Type                            | Functional Role                          | Input Node(s)           | Output Node(s)             | Sticky Note                                                                                              |
|--------------------------------|------------------------------------|----------------------------------------|------------------------|----------------------------|--------------------------------------------------------------------------------------------------------|
| Chat Trigger                   | Langchain Chat Trigger              | Receives user chat input                | External chat          | Agent: Topic + Research     | Receives user prompt from chat; greeting with quick-help text                                          |
| Agent: Topic + Research        | Langchain Agent                    | Analyzes topic and calls research tool | Chat Trigger           | Tool: Call Perplexity Researcher, Agent: Extract JSON | Drafts topic brief; calls research tool if needed; prompt tip to keep instructions short              |
| Tool: Call Perplexity Researcher| Langchain Tool Workflow            | Calls research sub-workflow (Perplexity) | Agent: Topic + Research | Agent: Extract JSON         | Set `workflowId` to research sub-workflow; outputs sources and takeaways                               |
| LM: For Topic Agent            | Langchain LM OpenAI (GPT-4o-mini) | Language model for Topic + Research agent | Agent: Topic + Research | Agent: Topic + Research     | Credentials configured externally                                                                     |
| Agent: Extract JSON            | Langchain Agent                    | Extracts structured JSON from text     | Agent: Topic + Research | Parser: Article JSON        | Extracts JSON; fill missing fields except quotes; declarative instructions                            |
| Parser: Article JSON           | Langchain Output Parser Structured | Validates article JSON structure        | Agent: Extract JSON     | Agent: Create Post Content  | Validates title, subtitle, content, hashtags; upstream agent must fill missing fields                 |
| LM: For JSON Extraction        | Langchain LM OpenAI (GPT-4o)       | Language model for JSON extraction      | Agent: Extract JSON     | Agent: Extract JSON         | Tune temperature if needed; credentials managed externally                                            |
| Agent: Create Post Content     | Langchain Agent                    | Converts article JSON to Vietnamese post| Parser: Article JSON    | Parser: Post Fields         | Creates concise post with emojis, CTAs; adjust tone/style in prompt                                   |
| LM: For Post Writer            | Langchain LM OpenAI (GPT-4o-mini) | Language model for post creation        | Agent: Create Post Content | Agent: Create Post Content | Lower temperature for consistency; credentials external                                              |
| Parser: Post Fields            | Langchain Output Parser Structured | Defines final social post fields        | Agent: Create Post Content | Publish: Facebook Graph API | Defines title, subtitle, content fields for post                                                     |
| Publish: Facebook Graph API    | n8n Facebook Graph API Node         | Publishes post to Facebook Page feed   | Parser: Post Fields     | None                       | Set Page ID and edge; disable during testing or add review step                                      |
| ⚙️ CONFIG (Set fields)          | Set Node                           | Holds configurable parameters           | None                   | All nodes via expressions   | Store Page ID, research workflow ID, emoji style, language; no sensitive data in nodes               |
| Sticky: Overview               | Sticky Note                       | Overview and instructions                | None                   | None                       | Workflow purpose, setup, requirements, customization, security notes                                 |
| Sticky: Chat Trigger           | Sticky Note                       | Chat Trigger explanation                 | None                   | None                       | Explains role and prompt design                                                                      |
| Sticky: Agent: Topic + Research | Sticky Note                       | Topic + Research agent explanation       | None                   | None                       | Advises on prompt brevity and research tool usage                                                    |
| Sticky: Tool: Call Perplexity Researcher | Sticky Note               | Research tool configuration advice      | None                   | None                       | Set `workflowId` before running                                                                      |
| Sticky: Parser: Article JSON   | Sticky Note                       | Article JSON parser description          | None                   | None                       | Validates article fields; upstream fill responsibility                                               |
| Sticky: Agent: Extract JSON    | Sticky Note                       | JSON extraction agent explanation        | None                   | None                       | Extraction instructions should be declarative                                                        |
| Sticky: Parser: Post Fields    | Sticky Note                       | Social post fields parser explanation    | None                   | None                       | Defines final post fields                                                                            |
| Sticky: Agent: Create Post Content | Sticky Note                   | Post creation agent guidance              | None                   | None                       | Emphasizes tone, emojis, CTA, Vietnamese language                                                   |
| Sticky: LM: For Topic Agent    | Sticky Note                       | Language model credential notes           | None                   | None                       | Credentials configured outside nodes                                                                |
| Sticky: LM: For JSON Extraction| Sticky Note                       | JSON extraction LM notes                  | None                   | None                       | Tune temperature, manage credentials externally                                                     |
| Sticky: LM: For Post Writer    | Sticky Note                       | Post writer LM notes                      | None                   | None                       | Use lower temperature for consistency                                                                |
| Sticky: Publish: Facebook Graph API | Sticky Note                  | Facebook API publishing guidance          | None                   | None                       | Configure Page ID and edge; disable for tests or add review                                         |
| Sticky: Template Checklist     | Sticky Note                       | Workflow quality checklist                 | None                   | None                       | Checklist for naming, notes, no hardcoded keys, config set, markdown description                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a `Set` node named "⚙️ CONFIG (Set fields)"**  
   - Add string fields:  
     - `FACEBOOK_PAGE_ID` = "YOUR_FACEBOOK_PAGE_ID"  
     - `FACEBOOK_EDGE` = "feed"  
     - `RESEARCH_WORKFLOW_ID` = "YOUR_RESEARCH_WORKFLOW_ID"  
     - `POST_EMOJI_STYLE` = "🔥 🚀 💬"  
     - `LANGUAGE` = "vi-VN"  
   - No inputs; outputs constants for use in expressions.

2. **Add a Langchain Chat Trigger node named "Chat Trigger"**  
   - Public access enabled.  
   - Initial messages:  
     ```
     Hi there! 👋
     I'm your Content Assistant. I can research with Perplexity and draft posts for you.
     How can I help today?
     ```  
   - Connect no inputs; this is entry point.

3. **Insert Langchain Agent node named "Agent: Topic + Research"**  
   - Prompt:  
     ```
     ={{$now}}
     Phân tích tự đề
     {{ $json.chatInput }}
     ```  
   - System message: instruct to use research tool if needed.  
   - Set prompt type to "define" with output parser enabled.  
   - Use OpenAI GPT-4o-mini model (configure credentials separately).  
   - Connect input from "Chat Trigger".

4. **Add Langchain Tool Workflow node named "Tool: Call Perplexity Researcher"**  
   - Set `workflowId` to your research sub-workflow ID (from CONFIG).  
   - Description: calls research tool to find up-to-date sources.  
   - Connect as a tool node invoked by "Agent: Topic + Research".

5. **Add Langchain Agent node named "Agent: Extract JSON"**  
   - Prompt:  
     ```
     Extract a JSON object from this content:{{ $json.output }}  and also fill this fields that is missing (exclude quote)
     ```  
   - Use OpenAI GPT-4o model (credentials externally configured).  
   - Enable output parser.  
   - Connect main input from "Agent: Topic + Research".

6. **Add Langchain Output Parser Structured node named "Parser: Article JSON"**  
   - Define manual JSON schema requiring:  
     - article: object with title, subtitle, content (mainText, sections with title, text, quote), hashtags (array of strings).  
   - Connect input from "Agent: Extract JSON".

7. **Add Langchain Agent node named "Agent: Create Post Content"**  
   - Prompt:  
     ```
     Convert this JSON into a concise and engaging blog in Vietnamese Language: {{$json.output.article.toJsonString() }}

     ## Formatting Guidelines
     - Write posts in a clear, natural tone
     - Use the title of the post as the opening line or focus.
     - Use emojis to highlight key ideas, e.g. 🌺 , l🔥 , 📣,
     - Keep content short and sweet, don't translate every word, but make sure the meaning flows and is easy to read.
     - Add compelling calls to action (CTA) such as: “See more now!”, “What do you think? Comment! 🚀”.
     - Use hashtags from the “hashtags” field in JSON to increase accessibility
     - Quotes or key ideas should be emphasized with quotation marks: “…” or add emoji: 💬 .
     - Avoid long sentences; keep the post easy to understand.
     - Make sure Vietnamese spelling and grammar are correct
     - Please do not use symbols such as ** or *
     ```  
   - Use OpenAI GPT-4o-mini model with output parser enabled.  
   - Connect input from "Parser: Article JSON".

8. **Add Langchain Output Parser Structured node named "Parser: Post Fields"**  
   - Provide example JSON schema with fields:  
     - title (string)  
     - subtitle (string)  
     - content (string)  
   - Connect input from "Agent: Create Post Content".

9. **Add n8n Facebook Graph API node named "Publish: Facebook Graph API"**  
   - Set Node = expression to `{{$json["FACEBOOK_PAGE_ID"] || "YOUR_FACEBOOK_PAGE_ID"}}` (or hardcode for setup).  
   - Set Edge = expression or "feed".  
   - HTTP method: POST.  
   - Query parameter: name = `message`, value = expression removing asterisks from final content: `={{ $json.content.replaceAll("*", "") }}`  
   - Connect input from "Parser: Post Fields".  
   - Ensure Facebook OAuth2 credentials are configured in n8n credentials.

10. **Configure Credentials**  
    - OpenAI API key with access to GPT-4o and GPT-4o-mini models.  
    - Facebook OAuth2 credentials with publish permissions on target Page.

11. **Test Flow**  
    - Start with "Chat Trigger" by sending a chat message.  
    - Verify research calls, JSON extraction, post creation, and optionally Facebook publishing.

12. **Optional**  
    - Link "Tool: Call Perplexity Researcher" to your research sub-workflow and set `RESEARCH_WORKFLOW_ID` in CONFIG node.  
    - Adjust prompts for tone, language, or platform variants.  
    - Insert moderation or review steps before publishing.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                       | Context or Link                                         |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------|
| This workflow is designed for marketers and content teams to quickly generate researched and polished social media posts from chat prompts, specifically targeting Facebook publishing.                                                             | Workflow overview sticky note                           |
| Before running, configure OpenAI and Facebook credentials securely in n8n, never hardcode API keys or sensitive IDs inside nodes.                                                                                                                | Security notes in CONFIG and sticky notes              |
| To use the research tool, link your own Perplexity research sub-workflow by setting its workflow ID in the "Tool: Call Perplexity Researcher" node.                                                                                                | Tool: Call Perplexity Researcher sticky note           |
| Consider disabling the Facebook publishing node during testing or routing outputs to a review step to avoid accidental posts.                                                                                                                    | Publish: Facebook Graph API sticky note                 |
| For better post quality, adjust the Vietnamese language prompts, emojis, and CTAs in the "Agent: Create Post Content" node prompt.                                                                                                                | Agent: Create Post Content sticky note                  |
| Use the checklist sticky note to ensure best practices: descriptive node names, use of sticky notes for documentation, no hardcoded keys, and markdown descriptions.                                                                              | Template Checklist sticky note                          |
| Workflow timezone is set to Asia/Bangkok; adjust as needed to your locale.                                                                                                                                                                         | n8n workflow settings                                   |

---

**Disclaimer:** The provided text and workflow are generated exclusively via an automated process using n8n, respecting all content policies and handling only legal, public data.