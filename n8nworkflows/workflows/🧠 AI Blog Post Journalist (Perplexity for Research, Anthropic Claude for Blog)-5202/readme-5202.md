🧠 AI Blog Post Journalist (Perplexity for Research, Anthropic Claude for Blog)

https://n8nworkflows.xyz/workflows/---ai-blog-post-journalist--perplexity-for-research--anthropic-claude-for-blog--5202


# 🧠 AI Blog Post Journalist (Perplexity for Research, Anthropic Claude for Blog)

---
### 1. Workflow Overview

This workflow, titled **"AI-Powered Blog Post Generator"**, automates the creation of SEO-optimized blog posts about the latest developments in artificial intelligence. It is designed for marketers, solo creators, and content teams who want to produce engaging, accessible AI-related content without manual research or writing.

**Target Use Cases:**  
- Daily or scheduled generation of fresh blog topics and posts related to AI  
- Translating complex AI research into easy-to-understand blog content for a general, non-technical audience  
- Automatic saving of finished blog posts into Google Docs for easy editing or publishing  
- Enhancing AI-generated content accuracy via memory and contextual research tools  

**Logical Blocks:**

- **1.1 Schedule Trigger:** Initiates the workflow on a recurring schedule (e.g., daily).  
- **1.2 AI Research:** Uses Perplexity AI to research and identify the most interesting recent AI news topics suitable for blog content.  
- **1.3 AI Writing Agent:** Employs Anthropic Claude via LangChain agent to convert research findings into a structured, SEO-optimized blog post tailored for non-technical readers.  
- **1.4 Memory & Research Tools:** Integrates a session-based memory buffer and Perplexity Search Tool to allow the AI agent to retrieve additional context or clarify facts during content generation.  
- **1.5 Output Storage:** Saves the generated blog post content directly into a specified Google Docs document for further use.

---

### 2. Block-by-Block Analysis

#### 2.1 Schedule Trigger

- **Overview:**  
  Kicks off the workflow on a recurring interval, enabling automation of the entire content generation process.

- **Nodes Involved:**  
  - Schedule Trigger

- **Node Details:**

  - **Schedule Trigger**  
    - *Type:* Schedule Trigger (n8n core node)  
    - *Configuration:* Runs on a default interval (daily by default if not further customized).  
    - *Key Expressions:* None; uses internal scheduling rules.  
    - *Input/Output:* No input; outputs trigger event to start workflow.  
    - *Version:* 1.2  
    - *Edge Cases:* Scheduling misconfiguration may prevent workflow execution. No direct failure modes beyond misfire.  
    - *Sub-workflow:* None

#### 2.2 AI Research

- **Overview:**  
  Utilizes Perplexity AI to research recent AI news from the past week, selecting the most interesting topics tailored for a non-technical blog audience.

- **Nodes Involved:**  
  - Perplexity - Blog Topic Research Node

- **Node Details:**

  - **Perplexity - Blog Topic Research Node**  
    - *Type:* Perplexity AI node (community node)  
    - *Configuration:*  
      - Model: "sonar-pro"  
      - Search Recency: Limited to the past week  
      - Message prompt: "Research on the latest AI news that will be interesting to blog about. Choose and return only the most interesting news for the blog intended for non-technical audience trying to learn more about AI."  
    - *Credentials:* Requires Perplexity API key  
    - *Input/Output:* Receives trigger from Schedule Trigger; outputs researched topic content under `.choices[0].message.content`.  
    - *Version:* 1  
    - *Edge Cases:* API authentication failure, rate limiting, or empty results if no relevant news found.  
    - *Sub-workflow:* None

#### 2.3 AI Writing Agent

- **Overview:**  
  Converts the AI research output into a complete, SEO-optimized blog post using Anthropic Claude via a LangChain AI agent node. It applies a detailed system prompt to ensure content suitability for general audiences and SEO best practices.

- **Nodes Involved:**  
  - AI Agent  
  - Anthropic Chat Model

- **Node Details:**

  - **Anthropic Chat Model**  
    - *Type:* LangChain Anthropic Chat Model node  
    - *Configuration:*  
      - Model: "claude-sonnet-4-20250514" (Claude 4 Sonnet)  
      - No additional options specified  
    - *Credentials:* Requires Anthropic API key  
    - *Input/Output:* Used as AI language model provider for the AI Agent node.  
    - *Version:* 1.3  
    - *Edge Cases:* API errors, model unavailability, or invalid API key.

  - **AI Agent**  
    - *Type:* LangChain AI Agent node  
    - *Configuration:*  
      - Text input: The content from Perplexity research node (`={{ $json.choices[0].message.content }}`)  
      - System prompt: Extensive instructions focusing on writing a friendly, clear, and SEO-friendly blog post for non-technical readers, including formatting rules, tone guidelines, and content structure (title, intro, sections, takeaway, meta description, SEO keywords).  
      - AI language model linked to Anthropic Chat Model node.  
      - AI tools linked to Perplexity Search Tool node for clarifications.  
      - AI memory linked to Simple Memory node for session context.  
    - *Input/Output:* Receives input from research node; outputs the full blog post content.  
    - *Version:* 2  
    - *Edge Cases:* Expression errors if input missing or malformed, API failures, ambiguity in source content, or memory retrieval errors.  
    - *Sub-workflow:* None

#### 2.4 Memory & Research Tools

- **Overview:**  
  Provides contextual memory and dynamic fact-checking capabilities to the AI Agent via a buffer memory and an integrated Perplexity Search Tool for on-demand clarifications.

- **Nodes Involved:**  
  - Simple Memory  
  - Perplexity Search Tool

- **Node Details:**

  - **Simple Memory**  
    - *Type:* LangChain Memory Buffer Window node  
    - *Configuration:*  
      - Session key: Custom key derived from the research node's output ID (`={{ $('Perplexity - Blog Topic Research Node').item.json.id }}`)  
      - Session ID type: Custom key  
    - *Input/Output:* Provides AI Agent with session-based memory context; input from research node, output to AI Agent.  
    - *Version:* 1.3  
    - *Edge Cases:* Missing or invalid session key may cause memory loss; memory overflow or mismatch could degrade response relevance.

  - **Perplexity Search Tool**  
    - *Type:* Perplexity Tool node  
    - *Configuration:*  
      - Model: "sonar-pro"  
      - Messages: Dynamically injected content from AI Agent's internal prompts (`={{ $fromAI('message0_Text', '', 'string') }}`)  
      - Simplify flag: Boolean flag injected dynamically (`={{ $fromAI('Simplify_Output', '', 'boolean') }}`)  
    - *Credentials:* Requires Perplexity API key  
    - *Input/Output:* Used as a tool by AI Agent for fact-checking or extended research during content generation.  
    - *Version:* 1  
    - *Edge Cases:* API limits, incorrect prompts leading to irrelevant results, or simplify flag misconfiguration.

#### 2.5 Output Storage

- **Overview:**  
  Saves the final AI-generated blog post text directly into a Google Docs document, enabling easy review, editing, or publishing workflows downstream.

- **Nodes Involved:**  
  - Google Docs

- **Node Details:**

  - **Google Docs**  
    - *Type:* Google Docs node (n8n core)  
    - *Configuration:*  
      - Operation: Update document  
      - Document URL: (Empty by default; must be filled with target Google Docs URL)  
      - Action: Insert text with the content from AI Agent output (`={{ $json.output }}`)  
    - *Credentials:* Google Docs OAuth2 account with write permissions  
    - *Input/Output:* Receives blog post text from AI Agent; writes content into Google Docs.  
    - *Version:* 2  
    - *Edge Cases:* Invalid or missing document URL, insufficient permissions, API rate limits, or connectivity issues.

---

### 3. Summary Table

| Node Name                         | Node Type                                   | Functional Role                     | Input Node(s)                      | Output Node(s)                      | Sticky Note                                                                                                           |
|----------------------------------|---------------------------------------------|-----------------------------------|----------------------------------|-----------------------------------|-----------------------------------------------------------------------------------------------------------------------|
| Schedule Trigger                 | Schedule Trigger (n8n core)                  | Initiates workflow on schedule    | None                             | Perplexity - Blog Topic Research Node | Schedule Trigger                                                                                                      |
| Perplexity - Blog Topic Research Node | Perplexity AI Node                          | Researches latest AI news topic   | Schedule Trigger                 | AI Agent                          | Schedule Trigger                                                                                                      |
| AI Agent                        | LangChain AI Agent Node                      | Generates blog post from research | Perplexity - Blog Topic Research Node, Simple Memory, Anthropic Chat Model, Perplexity Search Tool | Google Docs                        | Schedule Trigger                                                                                                      |
| Anthropic Chat Model            | LangChain Anthropic Chat Model Node          | Provides AI language model        | None (used by AI Agent)          | AI Agent                         | Schedule Trigger                                                                                                      |
| Simple Memory                   | LangChain Memory Buffer Window Node           | Provides session memory context   | Perplexity - Blog Topic Research Node | AI Agent                         | Schedule Trigger                                                                                                      |
| Perplexity Search Tool          | Perplexity Tool Node                          | Provides fact-checking & research | None (used by AI Agent)          | AI Agent                         | Schedule Trigger                                                                                                      |
| Google Docs                    | Google Docs Node (n8n core)                    | Saves blog post to Google Docs    | AI Agent                        | None                            | Schedule Trigger                                                                                                      |
| Sticky Note                    | Sticky Note Node                              | Annotation                      | None                             | None                            | 🧠 AI-Powered Blog Post Generator... [Full workflow description, usage, credit, and setup instructions]               |
| Sticky Note1                   | Sticky Note Node                              | Annotation                      | None                             | None                            | Schedule Trigger                                                                                                      |
| Sticky Note2                   | Sticky Note Node                              | Annotation                      | None                             | None                            | Schedule Trigger                                                                                                      |
| Sticky Note3                   | Sticky Note Node                              | Annotation                      | None                             | None                            | Schedule Trigger                                                                                                      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node:**  
   - Type: Schedule Trigger  
   - Set interval to your preferred frequency (e.g., daily).  
   - No credentials required.

2. **Add Perplexity - Blog Topic Research Node:**  
   - Type: Perplexity AI Node  
   - Credentials: Connect your Perplexity API key.  
   - Parameters:  
     - Model: sonar-pro  
     - Options: Set searchRecency to "week"  
     - Messages: Input prompt - "Research on the latest AI news that will be interesting to blog about. Choose and return only the most interesting news for the blog intended for non-technical audience trying to learn more about AI."  
   - Connect output of Schedule Trigger to this node’s input.

3. **Add Anthropic Chat Model node:**  
   - Type: LangChain Anthropic Chat Model node  
   - Credentials: Connect your Anthropic API key.  
   - Parameters:  
     - Model selection: Use "claude-sonnet-4-20250514" (Claude 4 Sonnet).  
   - No direct input connections (used internally by AI Agent).

4. **Add Simple Memory node:**  
   - Type: LangChain Memory Buffer Window node  
   - Parameters:  
     - Session Key: Use expression to reference research node’s output ID: `={{ $('Perplexity - Blog Topic Research Node').item.json.id }}`  
     - Session ID Type: Custom key  
   - Connect output of Perplexity Research node to this memory node.

5. **Add Perplexity Search Tool node:**  
   - Type: Perplexity Tool Node  
   - Credentials: Use Perplexity API key.  
   - Parameters:  
     - Model: sonar-pro  
     - Messages: Dynamic content injection from AI Agent internal prompts.  
     - Simplify flag: Also dynamically injected from AI Agent.  
   - No direct input connection; operates as AI Agent’s tool.

6. **Add AI Agent node:**  
   - Type: LangChain AI Agent node  
   - Parameters:  
     - Text: Reference Perplexity research output content: `={{ $json.choices[0].message.content }}`  
     - System Message: Use the detailed prompt that instructs to write a clear, SEO-optimized, friendly blog post structured with title, intro, sections, takeaway, and meta description, targeting a non-technical audience (copy prompt from node configuration).  
   - Connect inputs:  
     - Main input from Perplexity - Blog Topic Research Node  
     - AI Language Model input from Anthropic Chat Model node  
     - AI Memory input from Simple Memory node  
     - AI Tool input from Perplexity Search Tool node  
   - Output will be the generated blog post content.

7. **Add Google Docs node:**  
   - Type: Google Docs node  
   - Credentials: Connect Google Docs OAuth2 account with write access to your target doc.  
   - Parameters:  
     - Operation: Update  
     - Document URL: Paste the URL of your target Google Docs document.  
     - Actions: Insert the blog post content from AI Agent `={{ $json.output }}`  
   - Connect input from AI Agent output.

8. **Connect all nodes following the flow:**  
   - Schedule Trigger → Perplexity - Blog Topic Research Node → AI Agent → Google Docs  
   - Perplexity Research Node also connects to Simple Memory  
   - Simple Memory connects to AI Agent memory input  
   - Anthropic Chat Model connects to AI Agent language model input  
   - Perplexity Search Tool connects to AI Agent tool input

9. **Test and validate:**  
   - Ensure API keys are valid and connected.  
   - Validate Google Docs URL and permissions.  
   - Run workflow manually to verify blog post generation and saving.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   | Context or Link                                                |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------|
| 🧠 AI-Powered Blog Post Generator Category: Content Automation / AI Writing / Marketing Description: This automated workflow helps you generate fresh, SEO-optimized blog posts daily using AI tools—perfect for solo creators, marketers, and content teams looking to stay on top of the latest AI trends without manual research or writing. For more of this build and step-by-step tutorials, check out: https://www.youtube.com/@Automatewithmarc Here’s how it works: Schedule Trigger kicks off the workflow daily (or at your preferred interval). Perplexity AI Node researches the most interesting recent AI news tailored for a non-technical audience. AI Agent (Claude via Anthropic) turns that news into a full-length blog post based on a structured prompt that includes title, intro, 3+ section headers, takeaway, and meta description—designed for clarity, engagement, and SEO. Optional Memory & Perplexity Tool Nodes enhance the agent's responses by allowing it to clarify facts or fetch more context. Google Docs Node automatically saves the final blog post to your selected document—ready for review, scheduling, or publishing. Key Features: Combines Perplexity AI + Claude AI (Anthropic) for research + writing Built-in memory and retrieval logic for deeper contextual accuracy Non-technical, friendly writing style ideal for general audiences Output saved directly to Google Docs Fully no-code, customizable, and extendable Use Cases: Automate weekly blog content for your newsletter or site Repurpose content into social posts or scripts Keep your brand relevant in the fast-moving AI landscape Setup Requirements: Perplexity API Key Anthropic API Key Google Docs (OAuth2 connected) | Sticky Note on main canvas describing full workflow, features, and setup |

---

*Disclaimer: The content provided here is exclusively derived from an automated n8n workflow designed under compliance with content policies. It contains no illegal, offensive, or protected material, and all data processed are legal and publicly accessible.*