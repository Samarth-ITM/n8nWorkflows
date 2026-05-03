Classify and Convert GitHub Issues to Jira Tickets with OpenAI

https://n8nworkflows.xyz/workflows/classify-and-convert-github-issues-to-jira-tickets-with-openai-8216


# Classify and Convert GitHub Issues to Jira Tickets with OpenAI

### 1. Workflow Overview

This workflow automates the classification and conversion of newly created GitHub issues into corresponding Jira tickets, leveraging OpenAI's language model for intelligent issue classification. It is designed for teams that want to streamline issue tracking by automatically determining if an issue is a bug or a task and then creating the appropriate Jira ticket accordingly.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Detect new GitHub issues via a webhook trigger.
- **1.2 AI Processing:** Use an AI agent powered by OpenAI to analyze and classify the GitHub issue content.
- **1.3 Decision Making:** Evaluate AI output to determine if the issue is a bug or a task.
- **1.4 Jira Ticket Creation:** Create a Jira ticket based on the classification result.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block listens for new GitHub issues using a webhook, acting as the entry point to the workflow.

- **Nodes Involved:**  
  - GitHub Trigger

- **Node Details:**

  - **GitHub Trigger**  
    - *Type:* Trigger node for GitHub events.  
    - *Configuration:* Set to listen for new issues created in a configured GitHub repository.  
    - *Key expressions:* None explicitly; captures issue payload automatically.  
    - *Input/Output:* No input; outputs the GitHub issue data as JSON.  
    - *Version Requirements:* n8n version supporting GitHub Trigger node (v1+).  
    - *Potential Failures:* Webhook misconfiguration, permission errors on GitHub API, network timeouts.  
    - *Sub-workflow:* None.

#### 2.2 AI Processing

- **Overview:**  
  This block processes incoming GitHub issue data via an AI agent that uses OpenAI’s chat model and structured output parsing to classify the issue.

- **Nodes Involved:**  
  - AI Agent  
  - OpenAI Chat Model  
  - Simple Memory  
  - Structured Output Parser

- **Node Details:**

  - **AI Agent**  
    - *Type:* LangChain AI Agent node.  
    - *Configuration:* Integrates with OpenAI Chat Model, uses Simple Memory for context, and Structured Output Parser for extracting structured data.  
    - *Key expressions:* Receives raw issue data from GitHub Trigger, passes it to the OpenAI Chat Model; output parsed to derive classification.  
    - *Input/Output:* Input from GitHub Trigger; output to "Is it a bug?" decision node.  
    - *Version Requirements:* LangChain nodes v2.2.  
    - *Potential Failures:* API key authentication errors, AI response delays/timeouts, parsing failures if AI output is malformed.  
    - *Sub-workflow:* None.

  - **OpenAI Chat Model**  
    - *Type:* Language Model node, OpenAI Chat completion.  
    - *Configuration:* Uses OpenAI credentials and configured prompt for classification.  
    - *Key expressions:* Prompt designed to classify issue as "bug" or "task".  
    - *Input/Output:* Input to AI Agent node as language model; no direct external output.  
    - *Version Requirements:* n8n LangChain nodes v1.2+.  
    - *Potential Failures:* Quota limits, network errors, invalid API keys.

  - **Simple Memory**  
    - *Type:* LangChain Memory Buffer (windowed).  
    - *Configuration:* Maintains conversational context for AI agent to improve accuracy.  
    - *Key expressions:* Buffers recent interaction messages.  
    - *Input/Output:* Memory context input to AI Agent.  
    - *Version Requirements:* LangChain nodes v1.3+.  
    - *Potential Failures:* Memory overflow if improperly configured; negligible in this simple workflow.

  - **Structured Output Parser**  
    - *Type:* LangChain output parser.  
    - *Configuration:* Parses AI response into structured JSON to extract classification tag.  
    - *Key expressions:* Defines output schema for classification result.  
    - *Input/Output:* Input from OpenAI Chat Model; output to AI Agent node.  
    - *Version Requirements:* LangChain nodes v1.3+.  
    - *Potential Failures:* Parsing errors if AI output deviates from schema.

#### 2.3 Decision Making

- **Overview:**  
  This block routes the workflow based on the AI classification result—determining whether the issue is a bug or a task.

- **Nodes Involved:**  
  - Is it a bug?

- **Node Details:**

  - **Is it a bug?**  
    - *Type:* If node (conditional branching).  
    - *Configuration:* Checks AI Agent output field for classification—typically a boolean or string indicator of bug status.  
    - *Key expressions:* Evaluates AI output, e.g., `{{$json.classification === 'bug'}}`.  
    - *Input/Output:* Input from AI Agent; outputs to either "Create Jira Ticket (Bug)" or "Create Jira Ticket (Task)".  
    - *Version Requirements:* Standard n8n If node.  
    - *Potential Failures:* Expression errors if AI output is missing or malformed; logical errors if classification is ambiguous.

#### 2.4 Jira Ticket Creation

- **Overview:**  
  Creates Jira tickets in appropriate projects or issue types based on classification.

- **Nodes Involved:**  
  - Create Jira Ticket (Bug)  
  - Create Jira Ticket (Task)

- **Node Details:**

  - **Create Jira Ticket (Bug)**  
    - *Type:* Jira node (create issue).  
    - *Configuration:* Configured to create an issue of type "Bug" in a specified Jira project. Maps GitHub issue fields (title, description) accordingly.  
    - *Key expressions:* Uses AI-classified data and original issue details to populate fields.  
    - *Input/Output:* Input from "Is it a bug?" node's true branch; outputs no further nodes.  
    - *Version Requirements:* Jira node v1+. Requires Jira credentials with issue creation permissions.  
    - *Potential Failures:* Authentication errors, permission issues, field validation errors in Jira, network issues.

  - **Create Jira Ticket (Task)**  
    - *Type:* Jira node (create issue).  
    - *Configuration:* Similar to the Bug ticket node but uses issue type "Task".  
    - *Key expressions:* As above, adapted for Task issue type.  
    - *Input/Output:* Input from "Is it a bug?" node's false branch; outputs no further nodes.  
    - *Version Requirements:* Same as above.  
    - *Potential Failures:* Same as above.

---

### 3. Summary Table

| Node Name              | Node Type                            | Functional Role                      | Input Node(s)      | Output Node(s)                 | Sticky Note |
|------------------------|------------------------------------|------------------------------------|--------------------|-------------------------------|-------------|
| GitHub Trigger         | n8n-nodes-base.githubTrigger        | Triggers workflow on new GitHub issue | None               | AI Agent                      |             |
| AI Agent               | @n8n/n8n-nodes-langchain.agent      | Processes and classifies issue with AI | GitHub Trigger     | Is it a bug?                  |             |
| OpenAI Chat Model      | @n8n/n8n-nodes-langchain.lmChatOpenAi | Provides AI language model for classification | None (used by AI Agent) | AI Agent (languageModel input) |             |
| Simple Memory          | @n8n/n8n-nodes-langchain.memoryBufferWindow | Maintains AI conversation context | None (used by AI Agent) | AI Agent (memory input)        |             |
| Structured Output Parser | @n8n/n8n-nodes-langchain.outputParserStructured | Parses AI output into structured format | None (used by AI Agent) | AI Agent (outputParser input)  |             |
| Is it a bug?           | n8n-nodes-base.if                   | Branches flow based on bug classification | AI Agent           | Create Jira Ticket (Bug), Create Jira Ticket (Task) |             |
| Create Jira Ticket (Bug) | n8n-nodes-base.jira                 | Creates Jira bug ticket             | Is it a bug? (true) | None                         |             |
| Create Jira Ticket (Task) | n8n-nodes-base.jira                 | Creates Jira task ticket            | Is it a bug? (false) | None                         |             |
| Sticky Note            | n8n-nodes-base.stickyNote           | Annotation                        | None               | None                         |             |
| Sticky Note1           | n8n-nodes-base.stickyNote           | Annotation                        | None               | None                         |             |
| Sticky Note2           | n8n-nodes-base.stickyNote           | Annotation                        | None               | None                         |             |
| Sticky Note3           | n8n-nodes-base.stickyNote           | Annotation                        | None               | None                         |             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create GitHub Trigger Node:**  
   - Type: GitHub Trigger  
   - Configure webhook to listen for "Issues" events on your GitHub repository (specifically issue creation).  
   - Ensure GitHub credentials with repo access are set.

2. **Create OpenAI Chat Model Node:**  
   - Type: LangChain OpenAI Chat Model  
   - Configure with your OpenAI API key credentials.  
   - Set the prompt template to classify GitHub issues into "bug" or "task".  
   - Use suitable model (e.g., gpt-4 or gpt-3.5-turbo).

3. **Create Simple Memory Node:**  
   - Type: LangChain Memory Buffer Window  
   - Configure to retain recent messages for conversational context (default buffer size usually suffices).

4. **Create Structured Output Parser Node:**  
   - Type: LangChain Structured Output Parser  
   - Define a JSON schema output to parse AI responses, e.g.:  
     ```json
     { "classification": "bug" | "task" }
     ```  
   - This ensures AI output is structurally interpreted for downstream logic.

5. **Create AI Agent Node:**  
   - Type: LangChain AI Agent  
   - Connect inputs:  
       - `main` input from GitHub Trigger output  
       - `ai_languageModel` input from OpenAI Chat Model output  
       - `ai_memory` input from Simple Memory output  
       - `ai_outputParser` input from Structured Output Parser output  
   - Verify configuration to combine these elements for AI classification.

6. **Create "Is it a bug?" If Node:**  
   - Type: If node  
   - Set condition on AI Agent output, e.g.:  
     Expression: `{{$json["classification"] === "bug"}}`  
   - Connect AI Agent output to this node input.

7. **Create Jira Ticket (Bug) Node:**  
   - Type: Jira node (Create issue)  
   - Configure Jira credentials with issue creation permissions.  
   - Define project, issue type as "Bug".  
   - Map fields: title from GitHub issue title, description from issue body.  
   - Connect If node's true output to this node.

8. **Create Jira Ticket (Task) Node:**  
   - Type: Jira node (Create issue)  
   - Configure similarly but set issue type as "Task".  
   - Connect If node's false output to this node.

9. **Connect Workflow:**  
   - GitHub Trigger → AI Agent → Is it a bug? → Jira Ticket (Bug) or Jira Ticket (Task)

10. **Test and Validate:**  
    - Trigger a test GitHub issue creation and verify the correct Jira ticket is created according to the AI classification.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                             | Context or Link                                     |
|----------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------|
| Using AI for issue classification can improve triage accuracy but requires prompt tuning to avoid misclassification.                                     | AI prompt design best practices                     |
| Ensure Jira API credentials have sufficient permissions to create issues and access projects.                                                            | Jira API documentation: https://developer.atlassian.com/cloud/jira/platform/rest/v3/ |
| GitHub webhook must be properly configured on repository settings with correct URL and secret token for secure triggering.                              | GitHub Webhooks docs: https://docs.github.com/en/developers/webhooks-and-events/webhooks |
| For advanced setups, consider error handling nodes or retry logic around Jira ticket creation to handle API rate limits or transient failures.             | n8n error handling documentation                     |
| LangChain nodes require n8n version supporting these nodes (typically v0.213+). Use the latest n8n for best compatibility.                               | n8n node documentation: https://docs.n8n.io/nodes/ |
| This workflow example demonstrates integration of GitHub, OpenAI, and Jira APIs, showcasing powerful automation possibilities via n8n.                  |                                                    |

---

**Disclaimer:** The text above is an automated analysis generated from an n8n workflow configuration. It adheres strictly to all current content policies and contains no illegal or protected content. All referenced data and APIs are publicly accessible and legal.