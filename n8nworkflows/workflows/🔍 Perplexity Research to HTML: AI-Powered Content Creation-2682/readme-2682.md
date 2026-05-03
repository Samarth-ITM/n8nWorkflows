🔍 Perplexity Research to HTML: AI-Powered Content Creation

https://n8nworkflows.xyz/workflows/---perplexity-research-to-html--ai-powered-content-creation-2682


# 🔍 Perplexity Research to HTML: AI-Powered Content Creation

### 1. Workflow Overview

This n8n workflow, titled **"🔍 Perplexity Research to HTML: AI-Powered Content Creation"**, automates the transformation of user-submitted research queries into structured, publication-ready HTML content styled with Tailwind CSS. It integrates Perplexity AI for deep research and GPT-4 (via LangChain nodes) for content structuring and HTML generation. The workflow is designed to save time for content researchers, writers, and web publishers by automating research, content refinement, semantic structuring, and professional formatting.

The workflow comprises the following logical blocks:

- **1.1 Input Reception & Validation:** Handles incoming HTTP requests with user queries and validates the presence and quality of input.
- **1.2 Topic Enhancement:** Refines the user's initial query to optimize it for research.
- **1.3 Perplexity AI Research:** Performs automated research on the refined topic using Perplexity AI.
- **1.4 Article Extraction & Structuring:** Parses the research output into a structured article JSON.
- **1.5 Article to HTML Transformation:** Converts the structured article JSON into a clean, responsive HTML document with Tailwind CSS.
- **1.6 Response & Notifications:** Responds to the original webhook request with the final HTML content and optionally sends Telegram notifications.
- **1.7 Error Handling & No-Op Flows:** Handles cases of missing or invalid input gracefully.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception & Validation

- **Overview:**  
  Receives the GET request containing the research topic, extracts the query parameter, and validates its presence before proceeding.

- **Nodes Involved:**  
  - Webhook  
  - Get Topic  
  - If Topic Exists  
  - Do Nothing, do nothing (No-Op)  
  - Error Response  

- **Node Details:**  
  - **Webhook**  
    - Type: HTTP Webhook  
    - Role: Entry point; listens for GET requests at path `/pblog`.  
    - Config: Responds with node output; no authentication.  
    - Input: External HTTP GET request.  
    - Output: JSON containing query parameters.  
    - Failure cases: Missing query parameters or invalid HTTP request.

  - **Get Topic**  
    - Type: Set  
    - Role: Extracts `topic` field from the webhook query parameters (`$json.query.topic`).  
    - Config: Sets new field `topic` with extracted string.  
    - Input: Webhook output.  
    - Output: JSON enriched with `topic` field.  
    - Edge cases: `topic` may be empty or undefined if not provided.

  - **If Topic Exists**  
    - Type: If  
    - Role: Checks if `topic` is a non-empty string.  
    - Config: Condition `"notEmpty"` on `topic`.  
    - Input: Output of Get Topic.  
    - Output: Routes to topic improvement or No-Op/Error.  
    - Edge cases: Empty or missing topic leads to fallback.

  - **Do Nothing, do nothing (No-Op)**  
    - Type: No Operation  
    - Role: Pass-through for no topic scenario.  
    - Input: Negative branch from If Topic Exists.  
    - Output: Triggers Error Response.

  - **Error Response**  
    - Type: Set  
    - Role: Generates error message "Error. No topic provided."  
    - Output: JSON with error response string.  
    - Used when input is invalid or missing.

---

#### 1.2 Topic Enhancement

- **Overview:**  
  Improves the raw user topic by expanding and clarifying it, focusing on key concepts, components, applications, and insights to optimize research quality.

- **Nodes Involved:**  
  - Improve Users Topic (LLM Chain)  
  - If Topic (validation after improvement)  

- **Node Details:**  
  - **Improve Users Topic**  
    - Type: LangChain LLM Chain (OpenAI GPT-4o Mini)  
    - Role: Refines user topic prompt into a concise, enhanced 2-sentence prompt.  
    - Config: Prompt instructs focus on key concepts, definitions, core components, applications, analysis, and best practices.  
    - Input: `topic` string from previous block.  
    - Output: Improved topic string.  
    - Potential failures: OpenAI API errors, latency, or misinterpretation of prompt.

  - **If Topic**  
    - Type: If  
    - Role: Checks that improved topic text is not empty before proceeding.  
    - Output: Routes to Perplexity Topic Agent or No-Op.

---

#### 1.3 Perplexity AI Research

- **Overview:**  
  Uses the refined topic to run an AI-powered research agent querying Perplexity AI, retrieving detailed research outputs.

- **Nodes Involved:**  
  - Perplexity Topic Agent (LangChain Agent)  
  - Call Perplexity Researcher (Tool Workflow)  
  - Perplexity (HTTP Request)  
  - Success Response  
  - Telegram (notification)  
  - Chat Id (set Telegram chatId)  

- **Node Details:**  
  - **Perplexity Topic Agent**  
    - Type: LangChain Agent  
    - Role: Sends topic prompt to Perplexity research tool.  
    - Config: Uses system message to instruct use of `perplexity_research_tool`.  
    - Input: Improved topic string.  
    - Output: Research result text.  
    - Edge cases: API failures, empty or irrelevant output.

  - **Call Perplexity Researcher**  
    - Type: LangChain Tool Workflow  
    - Role: Invokes a sub-workflow named `perplexity_research_tool` with the topic as input.  
    - Sub-Workflow: "🔍🛠️Perplexity Researcher to HTML Web Page" (ID `HnqGW0eq5asKfZxf`)  
    - Input: Topic string.  
    - Output: Research output to Perplexity Topic Agent.  
    - Failure risks: Sub-workflow errors, improper input.

  - **Perplexity**  
    - Type: HTTP Request  
    - Role: Calls Perplexity API directly with system and user messages for chat completions.  
    - Config: POST request with JSON body containing model parameters, system, and user text.  
    - Authentication: Perplexity API key via HTTP Header Auth.  
    - Input: User and system prompt from earlier nodes.  
    - Output: Raw API response JSON.  
    - Failure modes: HTTP errors, auth failures, rate limiting.

  - **Success Response**  
    - Type: Set  
    - Role: Extracts content from Perplexity API response to a field `response`.  
    - Output: Prepares research text for downstream consumption.

  - **Telegram**  
    - Type: Telegram node  
    - Role: Sends a Telegram message with research snippet (first 300 chars).  
    - Config: Uses Telegram chat ID set earlier; HTML parse mode; no attribution.  
    - Input: Research output from Perplexity Topic Agent.  
    - Failure modes: Telegram API errors, invalid chat ID.

  - **Chat Id**  
    - Type: Set  
    - Role: Defines Telegram chat ID number for notifications (default 1234567890).  
    - Output: Enriches JSON with `telegram_chat_id` for notification nodes.

---

#### 1.4 Article Extraction & Structuring

- **Overview:**  
  Parses the raw Perplexity research output into a structured JSON schema representing the article, including metadata, sections, and hashtags.

- **Nodes Involved:**  
  - Extract JSON (LangChain Agent)  
  - Structured Output Parser1  
  - Article (Set)  
  - If Article  
  - Do Nothing2 (No-Op fallback)  

- **Node Details:**  
  - **Extract JSON**  
    - Type: LangChain Agent  
    - Role: Extracts JSON object from unstructured text output.  
    - Input: Research output text.  
    - Output: JSON with expected article fields.  
    - Retry on failure enabled.  
    - Failure cases: Parsing errors, malformed JSON.

  - **Structured Output Parser1**  
    - Type: LangChain Structured Output Parser  
    - Role: Defines manual JSON schema for article structure with required fields like category, title, metadata (author, timePosted, tag), content (mainText, sections with title, text, quote), and hashtags.  
    - Input: Raw AI output.  
    - Output: Validated structured JSON article.  
    - Failure modes: Schema validation errors.

  - **Article (Set)**  
    - Type: Set  
    - Role: Saves the parsed article JSON into `article` field for downstream use.  
    - Input: Output of Extract JSON.  
    - Output: Enriched JSON with article object.

  - **If Article**  
    - Type: If  
    - Role: Checks for non-empty `article` field before proceeding to HTML generation.  
    - Output: Routes to HTML agent or No-Op.

  - **Do Nothing2**  
    - Type: No Operation  
    - Role: Fallback for when no valid article JSON is parsed.

---

#### 1.5 Article to HTML Transformation

- **Overview:**  
  Converts the structured article JSON into a single-line, responsive HTML document enhanced with Tailwind CSS styling, ready for publication.

- **Nodes Involved:**  
  - Create HTML Article (LangChain Agent)  
  - If HTML  
  - Contents (Set)  
  - Basic LLM Chain (LangChain Chain LLM)  
  - Respond to Webhook  
  - Do Nothing3 (No-Op fallback)  
  - gpt-4o-mini, gpt-4o-mini1, gpt-4o-mini2, gpt-4o-mini3, gpt-4o-mini5 (OpenAI GPT-4o mini models)  
  - Sticky Notes (informational only)  

- **Node Details:**  
  - **Create HTML Article**  
    - Type: LangChain Agent  
    - Role: Converts article JSON into clean HTML using strict formatting rules (no `<html>`, `<body>`, `<style>`, `<head>` tags; uses `<h1>`, `<h2>`, `<p>`, `<blockquote>`; single line output).  
    - Input: Article JSON stringified.  
    - Output: JSON with `title` and `content` (HTML string).  
    - Retry on failure enabled.  
    - Failure risks: Formatting errors, LLM API issues.

  - **If HTML**  
    - Type: If  
    - Role: Validates presence of `title` and `content` in HTML output.  
    - Output: Routes to Contents node or No-Op.

  - **Contents (Set)**  
    - Type: Set  
    - Role: Extracts `title` and `content` from JSON string response for webhook response.  
    - Output: Enriches JSON with final HTML content.

  - **Basic LLM Chain**  
    - Type: LangChain Chain LLM  
    - Role: Takes article content and metadata to create a fully responsive, single-line HTML document with Tailwind CSS styling, enhanced layout with cards, proper list elements, and hashtags formatting.  
    - Input: Title, metadata, and raw HTML content from previous step.  
    - Output: Final HTML string optimized for SEO and responsiveness.  
    - Failure modes: API timeout, malformed HTML generation.

  - **Respond to Webhook**  
    - Type: Respond to Webhook  
    - Role: Sends final HTML string as plain text response to the initial HTTP request.  
    - Input: Output of Basic LLM Chain.  
    - Output: HTTP response to user.

  - **Do Nothing3**  
    - Type: No Operation  
    - Role: Fallback if HTML generation fails or content missing.

  - **GPT-4o Mini Nodes**  
    - Type: LangChain OpenAI LLM nodes  
    - Role: Various LLM calls to GPT-4o Mini model for topic improvement, JSON extraction, article creation, and final HTML generation.  
    - Credentials: OpenAI API credential named "OpenAi account".  
    - Potential issues: API limits, network errors, rate limiting.

  - **Sticky Notes**  
    - Informational only, visually organize workflow sections.

---

#### 1.6 Response & Notifications

- **Overview:**  
  Handles sending Telegram notifications with research snippets and final webhook response with the generated HTML content.

- **Nodes Involved:**  
  - Telegram2  
  - Telegram  
  - Chat Id, Chat Id1  
  - Respond to Webhook  

- **Node Details:**  
  - **Telegram2**  
    - Type: Telegram Node  
    - Role: Sends Telegram message with user raw topic input formatted in italic HTML.  
    - Input: Topic from Execute Workflow Trigger.  
    - Failure cases: Invalid chat ID or Telegram API failures.

  - **Telegram**  
    - See 1.3 Perplexity AI Research block.

  - **Chat Id, Chat Id1**  
    - Type: Set  
    - Role: Sets predefined Telegram chat IDs (default 1234567890).  
    - Used to configure Telegram messages.

  - **Respond to Webhook**  
    - See 1.5 Article to HTML Transformation block.

---

#### 1.7 Error Handling & No-Op Flows

- **Overview:**  
  Gracefully handles missing topics or empty outputs by short-circuiting workflow executions with no-ops or error responses.

- **Nodes Involved:**  
  - No Operation nodes (multiple: No Operation, do nothing; Do Nothing; Do Nothing1, 2, 3, 4)  
  - Error Response node  

- **Node Details:**  
  - All No-Op nodes serve as safe exit points that prevent workflow failures when conditions are unmet.  
  - Error Response provides a user-friendly error message when no topic is provided.

---

### 3. Summary Table

| Node Name                | Node Type                             | Functional Role                                     | Input Node(s)                | Output Node(s)               | Sticky Note                                                     |
|--------------------------|-------------------------------------|----------------------------------------------------|-----------------------------|-----------------------------|----------------------------------------------------------------|
| Webhook                  | HTTP Webhook                        | Receives initial GET request with user query       | —                           | Get Topic                   | ## Parse Topic from Get Request                                 |
| Get Topic                | Set                                | Extracts `topic` from query parameters              | Webhook                     | If Topic Exists             | ## Parse Topic from Get Request                                 |
| If Topic Exists          | If                                 | Checks if `topic` is non-empty                       | Get Topic                   | Improve Users Topic / No Op  | ## Parse Topic from Get Request                                 |
| Do Nothing, do nothing   | No Operation                       | Handles missing topic fallback                       | If Topic Exists             | Error Response              |                                                                |
| Error Response           | Set                                | Sets error response message                          | Do Nothing, do nothing      | —                           |                                                                |
| Improve Users Topic      | LangChain LLM Chain                | Refines and improves user topic prompt              | If Topic Exists (true)       | If Topic                   | ## Improve the Users Topic                                      |
| If Topic                 | If                                 | Validates improved topic text                        | Improve Users Topic          | Perplexity Topic Agent / No Op | ## Improve the Users Topic                                      |
| Perplexity Topic Agent   | LangChain Agent                   | Performs Perplexity AI research                      | If Topic                    | If2, Chat Id                | ## 🤖Perform Perplexity Research                                |
| If2                      | If                                 | Checks Perplexity AI output is non-empty            | Perplexity Topic Agent       | Extract JSON / Do Nothing   | ## 🤖Perform Perplexity Research                                |
| Chat Id                  | Set                                | Sets Telegram chat ID for notifications             | Perplexity Topic Agent       | Telegram                    |                                                                |
| Extract JSON             | LangChain Agent                   | Extracts structured JSON from research text         | If2                         | Article                     | ## Create Article from Perplexity Research                     |
| Article                  | Set                                | Stores structured article JSON                       | Extract JSON                | If Article                  | ## Create Article from Perplexity Research                     |
| If Article               | If                                 | Validates structured article presence                | Article                     | Create HTML Article / No Op | ## Create Article from Perplexity Research                     |
| Create HTML Article      | LangChain Agent                   | Converts article JSON to clean HTML content          | If Article                  | If HTML                     | ## Convert Article into HTML                                    |
| If HTML                  | If                                 | Checks HTML conversion output                         | Create HTML Article         | Contents / Do Nothing3      | ## Convert Article into HTML                                    |
| Contents                 | Set                                | Extracts title and HTML content from JSON response   | If HTML                     | Basic LLM Chain             | ## Convert Article into HTML                                    |
| Basic LLM Chain          | LangChain Chain LLM               | Generates final responsive HTML document with Tailwind CSS | Contents                   | Respond to Webhook          | ## Create HTML Page with TailwindCSS Styling                   |
| Respond to Webhook       | Respond to Webhook                | Sends final HTML content as HTTP response            | Basic LLM Chain             | —                           |                                                                |
| Telegram                 | Telegram Node                    | Sends Telegram notification with research snippet   | Chat Id                     | —                           |                                                                |
| Telegram2                | Telegram Node                    | Sends Telegram notification with raw user topic     | Chat Id1                    | If                         |                                                                |
| Chat Id1                 | Set                                | Sets Telegram chat ID for Telegram2                  | Execute Workflow Trigger    | Telegram2                   |                                                                |
| Execute Workflow Trigger | Execute Workflow Trigger         | Triggers workflow execution                           | —                           | Chat Id1                    | ## Optional                                                    |
| No Operation nodes       | No Operation                    | Multiple nodes to safely exit on invalid/missing data | Various                     | Various                     |                                                                |
| Call Perplexity Researcher | LangChain Tool Workflow          | Invokes sub-workflow for Perplexity research         | Perplexity Topic Agent       | Perplexity Topic Agent      |                                                                |
| Perplexity               | HTTP Request                    | Calls Perplexity API directly                          | Prompts                     | Success Response            |                                                                |
| Success Response         | Set                                | Extracts successful Perplexity API response           | Perplexity                  | —                           |                                                                |
| Prompts                  | Set                                | Sets system and user prompts for Perplexity API      | If                          | Perplexity                  |                                                                |
| Structured Output Parser1 | LangChain Output Parser Structured | Defines manual JSON schema for article extraction    | gpt-4o-mini2                | Extract JSON                | ## Create Article from Perplexity Research                     |
| Sticky Notes             | Sticky Note                      | Informational notes for workflow organization         | —                           | —                           | Multiple sticky notes visually group logical blocks            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**:  
   - Type: HTTP Webhook  
   - Path: `pblog`  
   - Method: GET  
   - Response Mode: `responseNode` (respond with node output)  
   - Purpose: Receive user query via GET parameter `topic`.

2. **Create Set Node "Get Topic"**:  
   - Assign `topic` = `{{$json.query.topic}}`  
   - Connect Webhook → Get Topic

3. **Create If Node "If Topic Exists"**:  
   - Condition: Check `topic` is not empty string (`notEmpty`)  
   - Connect Get Topic → If Topic Exists

4. **Create No Operation Node "Do Nothing, do nothing"**:  
   - Connect If Topic Exists (false) → Do Nothing, do nothing

5. **Create Set Node "Error Response"**:  
   - Set field `response` = "Error. No topic provided."  
   - Connect Do Nothing, do nothing → Error Response

6. **Create LangChain LLM Chain Node "Improve Users Topic"**:  
   - Model: GPT-4o Mini  
   - Prompt: Instruction to improve user topic focusing on key concepts, core components, applications, analysis, best practices (max 2 sentences)  
   - Connect If Topic Exists (true) → Improve Users Topic

7. **Create If Node "If Topic"**:  
   - Condition: Improved topic text is not empty  
   - Connect Improve Users Topic → If Topic

8. **Create LangChain Agent Node "Perplexity Topic Agent"**:  
   - System Message: "Use the perplexity_research_tool to provide research on the users topic."  
   - Input: Topic string from If Topic (true)  
   - Connect If Topic (true) → Perplexity Topic Agent

9. **Create LangChain Tool Workflow Node "Call Perplexity Researcher"**:  
   - Tool Workflow: Use sub-workflow ID `HnqGW0eq5asKfZxf` named `perplexity_research_tool`  
   - Input Field: `topic` string from Perplexity Topic Agent  
   - Connect Perplexity Topic Agent → Call Perplexity Researcher  
   - Connect Call Perplexity Researcher → Perplexity Topic Agent (for tool output)

10. **Create If Node "If2"**:  
    - Condition: Check Perplexity Topic Agent output `output` is not empty  
    - Connect Perplexity Topic Agent → If2

11. **Create LangChain Agent Node "Extract JSON"**:  
    - Prompt: Extract JSON object from Perplexity output text  
    - Retry on Fail: Enabled  
    - Connect If2 (true) → Extract JSON

12. **Create LangChain Structured Output Parser "Structured Output Parser1"**:  
    - Define manual JSON schema for article with required fields (category, title, metadata, content sections, hashtags)  
    - Connect GPT-4o Mini2 (LLM node) → Structured Output Parser1  
    - Connect Structured Output Parser1 → Extract JSON

13. **Create Set Node "Article"**:  
    - Assign `article` = `{{$json.output.article}}`  
    - Connect Extract JSON → Article

14. **Create If Node "If Article"**:  
    - Condition: `article` field is non-empty string  
    - Connect Article → If Article

15. **Create LangChain Agent Node "Create HTML Article"**:  
    - Prompt: Convert article JSON to HTML with strict formatting rules (single line, use `<h1>`, `<h2>`, `<p>`, `<blockquote>`, no `<html>`, `<body>`, etc.)  
    - Retry on Fail: Enabled  
    - Connect If Article (true) → Create HTML Article

16. **Create If Node "If HTML"**:  
    - Condition: Check `title` and `content` exist and are non-empty in Create HTML Article output JSON  
    - Connect Create HTML Article → If HTML

17. **Create Set Node "Contents"**:  
    - Assign `title` and `content` from parsed JSON output of Create HTML Article  
    - Connect If HTML (true) → Contents

18. **Create LangChain Chain LLM Node "Basic LLM Chain"**:  
    - Prompt: Create a modern, responsive, single-line HTML document with Tailwind CSS, enhanced styling, and proper semantic tags, using `title`, `metadata`, and HTML content from Contents node  
    - Connect Contents → Basic LLM Chain

19. **Create Respond to Webhook Node**:  
    - Response Type: Text  
    - Response Body: `{{$json.text}}` (final HTML string)  
    - Connect Basic LLM Chain → Respond to Webhook

20. **Create Telegram Notification Nodes (Optional)**:  
    - Set Nodes "Chat Id" and "Chat Id1" with Telegram chat IDs  
    - Telegram Node: Sends research snippet (first 300 chars from Perplexity Topic Agent output)  
    - Telegram2 Node: Sends italicized raw user topic (from Execute Workflow Trigger)  
    - Connect accordingly for notification flows

21. **Create No-Op Nodes for Negative Branches**:  
    - Add No Operation nodes for branches where conditions fail (missing topic, empty outputs, etc.) to safely terminate flows without errors.

22. **Configure Credentials**:  
    - OpenAI API credential configured for all GPT-4o Mini LangChain nodes  
    - Perplexity API key for HTTP Request node (Perplexity)  
    - Telegram API credentials for Telegram nodes

23. **Sub-Workflow Setup** (`perplexity_research_tool`):  
    - Confirm sub-workflow with ID `HnqGW0eq5asKfZxf` is imported and configured correctly.  
    - Expects input parameter: `topic` (string)  
    - Outputs research text for Perplexity Topic Agent.

---

### 5. General Notes & Resources

| Note Content                                                                                                            | Context or Link                                                  |
|-------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------|
| Workflow uses Tailwind CSS v2.2.19 CDN for styling the generated HTML pages.                                              | See Basic LLM Chain prompt example in node configuration.       |
| Telegram nodes require a valid Telegram Bot API token and chat ID to send notifications.                                 | Telegram Developer Bot API documentation                        |
| The workflow tightly integrates with Perplexity AI via HTTP requests and LangChain agents for rich research retrieval.   | Perplexity API docs: https://api.perplexity.ai                   |
| OpenAI GPT-4o Mini model is used for all LLM tasks, including prompt improvement, JSON extraction, article generation.   | Requires OpenAI API key with access to GPT-4o Mini               |
| Single line HTML output with no newlines (except `</br>`) ensures compatibility with text-based output channels.        | Ensures clean pasting into web CMS or direct HTML injection.    |
| Sticky Notes are used extensively to logically separate workflow blocks and provide context within the editor UI.       | Visual guidance only, no runtime impact.                         |

---

This comprehensive documentation enables developers and AI agents to fully understand, reproduce, and extend the workflow with clarity and precision. It anticipates potential edge cases and integration issues, especially around API calls and content parsing.