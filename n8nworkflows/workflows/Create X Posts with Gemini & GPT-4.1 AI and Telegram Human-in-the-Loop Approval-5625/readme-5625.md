Create X Posts with Gemini & GPT-4.1 AI and Telegram Human-in-the-Loop Approval

https://n8nworkflows.xyz/workflows/create-x-posts-with-gemini---gpt-4-1-ai-and-telegram-human-in-the-loop-approval-5625


# Create X Posts with Gemini & GPT-4.1 AI and Telegram Human-in-the-Loop Approval

### 1. Workflow Overview

This workflow automates the creation and publishing of multiple posts on Twitter (X) using advanced AI models (Gemini & GPT-4.1) combined with human-in-the-loop approval via Telegram. The process includes content generation, classification, revision, and final posting stages with feedback mechanisms integrated for quality control.

Logical blocks:

- **1.1 Input Reception:** Receives post creation requests through Telegram.
- **1.2 AI Content Generation:** Uses AI agents and language models (Gemini and GPT-4.1) to generate and refine post content.
- **1.3 Content Classification:** Classifies generated content to decide publishing readiness or revision needs.
- **1.4 Human Feedback Loop:** Sends generated content to Telegram for human review and feedback.
- **1.5 Revision and Finalization:** Revises posts based on classification and feedback, preparing final content for publishing.
- **1.6 Publishing:** Posts the approved content to Twitter (X).
- **1.7 Auxiliary Tools:** Includes external HTTP tool integration for additional data sourcing (Tavily Search).

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:** Listens for new post requests from Telegram and triggers the workflow.
- **Nodes Involved:** Telegram_Trigger
- **Node Details:**

  - **Telegram_Trigger**
    - Type: Telegram Trigger node
    - Role: Listens for incoming Telegram messages to initiate the workflow.
    - Configuration: Uses a webhook ID configured with Telegram Bot API.
    - Inputs: External Telegram user messages.
    - Outputs: Passes received message data to the AI content generation block.
    - Failures: Possible webhook misconfiguration, Telegram API downtime, or message format errors.

#### 1.2 AI Content Generation

- **Overview:** Generates initial post content using AI agents powered by Gemini and GPT-4.1 language models.
- **Nodes Involved:** X Post Agent, GPT 4.1, 2.0 Flash, Tavily Search
- **Node Details:**

  - **X Post Agent**
    - Type: LangChain AI Agent node
    - Role: Coordinates AI tools to generate post content.
    - Configuration: Uses Gemini and GPT-4.1 models, incorporates HTTP tool (Tavily Search) as an auxiliary resource.
    - Inputs: Triggered by Telegram_Trigger output.
    - Outputs: Generated content data sent to Set Post node for structuring.
    - Edge Cases: API key or quota issues, timeout from AI service, incomplete data generation.

  - **GPT 4.1**
    - Type: LangChain OpenAI Chat Router node
    - Role: Provides GPT-4.1 language model capabilities for content generation.
    - Configuration: Connected as AI language model backend for X Post Agent and Revision Agent nodes.
    - Inputs: AI prompt requests from agents.
    - Outputs: Text completions and suggestions.
    - Failures: OpenAI API authentication or rate limits, malformed prompts.

  - **2.0 Flash**
    - Type: LangChain OpenAI Chat Router node
    - Role: Alternative AI model (probably Gemini or flash version) used for classification tasks.
    - Configuration: Supports Text Classifier node.
    - Inputs: AI prompt requests for classification.
    - Outputs: Classification results.
    - Failures: Similar to GPT 4.1 node.

  - **Tavily Search**
    - Type: LangChain HTTP Request Tool node
    - Role: External HTTP API integration to fetch additional data for post generation.
    - Configuration: HTTP request tool configured with Tavily API or similar.
    - Inputs: Requests from X Post Agent for enrichment.
    - Outputs: External data to aid AI content generation.
    - Edge Cases: HTTP errors, invalid API credentials, network issues.

#### 1.3 Content Classification

- **Overview:** Classifies AI-generated content to decide whether it is ready for publishing or needs revision.
- **Nodes Involved:** Text Classifier
- **Node Details:**

  - **Text Classifier**
    - Type: LangChain textClassifier node
    - Role: Analyzes generated post content to determine quality and appropriateness.
    - Configuration: Uses AI models via 2.0 Flash node.
    - Inputs: Content from Request Feedback node.
    - Outputs: Classification results direct content either for publishing or revision.
    - Failures: Misclassification risks, AI model errors, or input data issues.

#### 1.4 Human Feedback Loop

- **Overview:** Sends generated posts to a human reviewer on Telegram for approval or feedback.
- **Nodes Involved:** Request Feedback, Set Post
- **Node Details:**

  - **Set Post**
    - Type: Set node
    - Role: Formats and prepares post content for Telegram feedback request.
    - Configuration: Sets key variables such as post text and metadata.
    - Inputs: Output from AI agents and revision nodes.
    - Outputs: Content sent to Request Feedback node.
    - Failures: Expression evaluation errors or missing data.

  - **Request Feedback**
    - Type: Telegram node
    - Role: Sends message to Telegram chat requesting human review.
    - Configuration: Configured with Telegram Bot credentials.
    - Inputs: Formatted post from Set Post.
    - Outputs: User feedback collected and passed to classifier.
    - Failures: Telegram API errors, message delivery failure.

#### 1.5 Revision and Finalization

- **Overview:** Revises post content based on classification and human feedback, then finalizes it.
- **Nodes Involved:** Revision Agent, Set Post (same as feedback block)
- **Node Details:**

  - **Revision Agent**
    - Type: LangChain agent node
    - Role: Uses GPT-4.1 to revise or enhance posts flagged for changes.
    - Configuration: Uses GPT 4.1 as language model backend.
    - Inputs: Posts flagged by classification or feedback.
    - Outputs: Revised posts forwarded for final preparation.
    - Failures: Revision failures due to AI errors or inconsistent input.

#### 1.6 Publishing

- **Overview:** Publishes the approved and finalized post to Twitter (X).
- **Nodes Involved:** X Post
- **Node Details:**

  - **X Post**
    - Type: Twitter node
    - Role: Posts the final content to Twitter (X).
    - Configuration: Uses Twitter OAuth2 credentials.
    - Inputs: Approved content from Text Classifier or Revision Agent.
    - Outputs: Confirmation of posted tweet.
    - Failures: Twitter API authentication, rate limits, content length limits.

#### 1.7 Auxiliary Tools and Notes

- **Sticky Notes:** Various sticky notes exist for comments/notes but contain no content.
- No sub-workflows or external workflow calls detected.

---

### 3. Summary Table

| Node Name         | Node Type                       | Functional Role                    | Input Node(s)        | Output Node(s)           | Sticky Note |
|-------------------|--------------------------------|----------------------------------|----------------------|--------------------------|-------------|
| Telegram_Trigger   | Telegram Trigger                | Receive Telegram input            | -                    | X Post Agent             |             |
| X Post Agent      | LangChain Agent                | Generate initial posts            | Telegram_Trigger, Tavily Search, GPT 4.1 | Set Post                  |             |
| GPT 4.1           | LangChain LM Chat Open Router  | Provide GPT-4.1 language model   | X Post Agent, Revision Agent | Revision Agent, X Post Agent |             |
| 2.0 Flash         | LangChain LM Chat Open Router  | Provide alternative AI model     | -                    | Text Classifier          |             |
| Tavily Search     | LangChain HTTP Request Tool    | External HTTP data fetch          | -                    | X Post Agent             |             |
| Set Post          | Set                            | Format post content               | X Post Agent, Revision Agent | Request Feedback         |             |
| Request Feedback  | Telegram                       | Request human feedback on Telegram | Set Post             | Text Classifier          |             |
| Text Classifier   | LangChain Text Classifier      | Classify post content             | Request Feedback, 2.0 Flash | X Post, Revision Agent   |             |
| Revision Agent    | LangChain Agent                | Revise posts based on feedback   | Text Classifier, GPT 4.1 | Set Post                  |             |
| X Post            | Twitter                       | Post content to Twitter (X)      | Text Classifier       | -                        |             |
| Sticky Note       | Sticky Note                   | Comments/Notes                   | -                    | -                        |             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger Node**
   - Type: Telegram Trigger
   - Configure with Telegram Bot credentials.
   - Set webhook to receive messages.
   - No parameters needed for basic trigger.

2. **Create LangChain Agent Node "X Post Agent"**
   - Type: LangChain Agent
   - Configure to use Gemini and GPT-4.1 via linked LM Chat Router nodes.
   - Add Tavily Search as an HTTP tool for enrichment.
   - Connect Telegram Trigger output to this node’s input.

3. **Create GPT 4.1 LangChain LM Chat Open Router Node**
   - Type: LM Chat Open Router
   - Configure with OpenAI GPT-4 API credentials.
   - Connect as AI language model backend for X Post Agent and Revision Agent nodes.

4. **Create 2.0 Flash LangChain LM Chat Open Router Node**
   - Type: LM Chat Open Router
   - Configure with appropriate credentials (likely Gemini API or OpenAI).
   - Connect as AI model for Text Classifier node.

5. **Create Tavily Search LangChain HTTP Request Tool Node**
   - Type: HTTP Request Tool
   - Configure with Tavily API credentials or relevant HTTP endpoint.
   - Connect output to X Post Agent as an auxiliary tool.

6. **Create Set Node "Set Post"**
   - Type: Set
   - Configure to format and prepare post text and metadata for Telegram feedback.
   - Connect output of X Post Agent and Revision Agent to Set Post input.

7. **Create Telegram Node "Request Feedback"**
   - Type: Telegram
   - Configure with Telegram Bot credentials.
   - Set parameters to send message with post content for human review.
   - Connect Set Post output to this node.

8. **Create LangChain Text Classifier Node**
   - Type: Text Classifier
   - Connect output of Request Feedback (user feedback messages).
   - Use 2.0 Flash node as AI language model backend.
   - Connect output to both X Post node and Revision Agent node based on classification.

9. **Create LangChain Agent Node "Revision Agent"**
   - Type: LangChain Agent
   - Configure with GPT 4.1 as language model backend.
   - Connect input from Text Classifier’s revision path.
   - Output connects back to Set Post for final formatting.

10. **Create Twitter Node "X Post"**
    - Type: Twitter
    - Configure with Twitter OAuth2 credentials.
    - Connect input from Text Classifier’s approved content path.
    - Setup for posting tweets with text content.

11. **Connect all nodes as per the logical flow:**
    - Telegram_Trigger → X Post Agent
    - Tavily Search → X Post Agent (tool)
    - X Post Agent → Set Post → Request Feedback → Text Classifier
    - Text Classifier → Revision Agent (if revision needed)
    - Revision Agent → Set Post → Request Feedback (loop if necessary)
    - Text Classifier → X Post (if approved)

---

### 5. General Notes & Resources

| Note Content                                                                                       | Context or Link                  |
|--------------------------------------------------------------------------------------------------|--------------------------------|
| The workflow integrates advanced AI models Gemini & GPT-4.1 to enhance content creativity.        | Workflow title and AI nodes     |
| Human-in-the-loop approval via Telegram ensures quality and compliance before posting to Twitter.| Telegram feedback nodes         |
| External data enrichment via Tavily Search enhances AI-generated content with relevant info.      | Tavily Search HTTP tool         |
| Twitter API integration requires OAuth2 credentials with necessary permissions for posting tweets.| Twitter node configuration      |
| This workflow is designed for multi-post creation with iterative revision and classification.     | Overall workflow purpose        |

---

**Disclaimer:**  
The provided content originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.