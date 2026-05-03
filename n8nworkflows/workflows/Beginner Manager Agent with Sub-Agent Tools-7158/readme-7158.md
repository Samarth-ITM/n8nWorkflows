Beginner Manager Agent with Sub-Agent Tools

https://n8nworkflows.xyz/workflows/beginner-manager-agent-with-sub-agent-tools-7158


# Beginner Manager Agent with Sub-Agent Tools

### 1. Workflow Overview

This workflow titled **"Beginner Manager Agent with Sub-Agent Tools"** is designed to orchestrate a manager AI agent that delegates user queries intelligently to specialized sub-agents. The manager analyzes input messages and decides whether the task involves writing emails or performing data analysis, routing the query accordingly. It combines language models, memory buffers, and agent tools to create a modular, extensible AI assistant framework.

**Target use cases:**  
- Automatically determining the nature of incoming user requests (communication vs. data insights)  
- Generating professional emails for outreach, follow-ups, or client communication  
- Producing metrics, summaries, or data analysis explanations on demand  
- Demonstrating a clear pattern for building multi-agent systems with delegation  

**Logical blocks:**

- **1.1 Manager Agent Setup**: Receives user input, analyzes it, and routes instructions to specialized sub-agents.  
- **1.2 Email Agent Tool**: Handles requests related to writing polished, professional emails.  
- **1.3 Data Agent Tool**: Handles requests related to data summaries, metrics, and analysis explanations.  
- **1.4 Memory Buffers**: Maintains conversation context for each agent to provide relevant and coherent responses.  
- **1.5 Language Model Nodes**: Provide the underlying GPT-4o-mini model for all agents to generate responses.  
- **1.6 Sticky Notes**: Documentation nodes embedded in the workflow for user guidance.

---

### 2. Block-by-Block Analysis

#### 1.1 Manager Agent Setup

**Overview:**  
This is the core routing agent that receives user queries and decides which specialized sub-agent (EmailAgent or DataAgent) should handle the task. It sends instructions to these sub-agents based on an internal system message describing the roles.

**Nodes Involved:**  
- ManagerAgent (Routing + Instruction Generator)  
- Simple Memory2  
- OpenAI Chat Model  
- Connections to EmailAgent and DataAgent tools

**Node Details:**  

- **ManagerAgent (Routing + Instruction Generator)**  
  - *Type:* Langchain Agent Node  
  - *Role:* AI Manager that delegates tasks to sub-agents  
  - *Configuration:*  
    - System Message instructs it to analyze the user's message to decide between EmailAgent and DataAgent.  
    - Sends instructions to sub-agents accordingly.  
  - *Expressions:* Uses input from connected nodes (e.g., user query) implicitly.  
  - *Inputs:* Receives AI language model output and AI memory.  
  - *Outputs:* Sends delegated instructions to EmailAgent and DataAgent nodes via `ai_tool` connections.  
  - *Failure types:* Potential issues include model API failures, misrouting due to ambiguous instructions, or memory buffer inconsistencies.  
  - *Version:* 2.2

- **Simple Memory2**  
  - *Type:* Langchain Buffer Memory Window  
  - *Role:* Maintains conversational memory for the ManagerAgent to preserve context and history.  
  - *Configuration:* Defaults (window buffer) to keep recent conversation turns.  
  - *Inputs:* Takes output from ManagerAgent.  
  - *Outputs:* Feeds memory context back to ManagerAgent for context-aware decision-making.  
  - *Failure types:* Memory overflow or loss of context if buffer size is too small.

- **OpenAI Chat Model**  
  - *Type:* Langchain OpenAI GPT Chat Model  
  - *Role:* Provides GPT-4o-mini LLM responses for the ManagerAgent.  
  - *Configuration:* Model set to "gpt-4o-mini" with default options.  
  - *Credentials:* Uses OpenAI API credential "OpenAi account 4."  
  - *Inputs:* Feeds into ManagerAgent agent node.  
  - *Failure types:* API key invalid, rate limits, or network issues.  
  - *Version:* 1.2

---

#### 1.2 Email Agent Tool

**Overview:**  
A specialized sub-agent tool dedicated to generating professional emails for outreach, follow-ups, or templated communication based on instructions from the ManagerAgent.

**Nodes Involved:**  
- EmailAgent (Communication Specialist)  
- Simple Memory  
- OpenAI Chat Model1  

**Node Details:**  

- **EmailAgent (Communication Specialist)**  
  - *Type:* Langchain Agent Tool Node  
  - *Role:* Writes professional, friendly, or action-oriented emails as instructed.  
  - *Configuration:*  
    - Input text uses expression: `{{ $fromAI('Prompt__User_Message_', ``, 'string') }}` to receive instructions from ManagerAgent.  
    - System Message guides tone and style: warm, business-appropriate, polished emails.  
    - Tool Description explicitly states its email-writing role.  
  - *Inputs:* Receives instructions from ManagerAgent via `ai_tool` input.  
  - *Outputs:* Returns generated email content back to ManagerAgent.  
  - *Failure types:* Poor prompt interpretation, incorrect tone, or model API errors.  
  - *Version:* 2.2

- **Simple Memory**  
  - *Type:* Langchain Buffer Memory Window  
  - *Role:* Maintains conversation memory for the EmailAgent to keep track of email drafts or iterative instructions.  
  - *Configuration:* Default buffer window.  
  - *Inputs/Outputs:* Works bi-directionally with EmailAgent.  
  - *Failure types:* Context loss or memory buffer overflow.

- **OpenAI Chat Model1**  
  - *Type:* Langchain OpenAI GPT Chat Model  
  - *Role:* Supplies GPT-4o-mini LLM responses for EmailAgent.  
  - *Configuration:* Same model as others, default options.  
  - *Credentials:* Uses same OpenAI API credential.  
  - *Failure types:* API issues or latency.  
  - *Version:* 1.2

---

#### 1.3 Data Agent Tool

**Overview:**  
This sub-agent specializes in handling requests that require data insights, metric analysis, and summary generation. It processes queries delegated by the ManagerAgent and returns relevant data-driven responses.

**Nodes Involved:**  
- DataAgent (Insight Generator)  
- Simple Memory1  
- OpenAI Chat Model2  

**Node Details:**  

- **DataAgent (Insight Generator)**  
  - *Type:* Langchain Agent Tool Node  
  - *Role:* Responds to instructions requiring metrics, summaries, or data analysis explanations.  
  - *Configuration:*  
    - Input text set to `{{json.query}}` referencing the delegated query.  
    - Tool Description clarifies its analytical focus.  
  - *Inputs:* Receives query instructions from ManagerAgent via `ai_tool` input.  
  - *Outputs:* Returns analyzed insights to ManagerAgent.  
  - *Failure types:* Misinterpretation of data requests, model errors, or input data formatting issues.  
  - *Version:* 2.2

- **Simple Memory1**  
  - *Type:* Langchain Buffer Memory Window  
  - *Role:* Maintains conversation context for DataAgent.  
  - *Configuration:* Default memory buffer.  
  - *Inputs/Outputs:* Supports DataAgent for contextual continuity.  
  - *Failure types:* Similar to other memory nodes.

- **OpenAI Chat Model2**  
  - *Type:* Langchain OpenAI GPT Chat Model  
  - *Role:* Provides GPT-4o-mini LLM for data analysis responses.  
  - *Configuration:* Same as others.  
  - *Credentials:* Same OpenAI API credential.  
  - *Failure types:* Usual API or network failures.  
  - *Version:* 1.2

---

#### 1.4 Sticky Notes (Documentation)

**Overview:**  
These nodes provide embedded documentation, instructions, and helpful contact information within the workflow for users setting up or customizing the system.

**Nodes Involved:**  
- Sticky Note16  
- Sticky Note  
- Sticky Note1  
- Sticky Note2  

**Details:**  
- Contain step-by-step setup instructions for ManagerAgent, EmailAgent, and DataAgent.  
- Provide contact info and social media links for support.  
- Positioned to offer guidance alongside workflow structure.

---

### 3. Summary Table

| Node Name                          | Node Type                                | Functional Role                                  | Input Node(s)                          | Output Node(s)                         | Sticky Note                                                                                                  |
|-----------------------------------|-----------------------------------------|-------------------------------------------------|--------------------------------------|--------------------------------------|--------------------------------------------------------------------------------------------------------------|
| OpenAI Chat Model                 | Langchain OpenAI Chat Model              | Provides LLM responses for ManagerAgent          | —                                    | ManagerAgent (Routing + Instruction Generator) |                                                                                                              |
| Simple Memory                    | Langchain Memory Buffer Window           | Memory for EmailAgent                             | EmailAgent (Communication Specialist) | EmailAgent (Communication Specialist) |                                                                                                              |
| Simple Memory1                   | Langchain Memory Buffer Window           | Memory for DataAgent                              | DataAgent (Insight Generator)         | DataAgent (Insight Generator)         |                                                                                                              |
| OpenAI Chat Model1               | Langchain OpenAI Chat Model              | Provides LLM for EmailAgent                       | —                                    | EmailAgent (Communication Specialist) |                                                                                                              |
| OpenAI Chat Model2               | Langchain OpenAI Chat Model              | Provides LLM for DataAgent                        | —                                    | DataAgent (Insight Generator)         |                                                                                                              |
| DataAgent (Insight Generator)    | Langchain Agent Tool                     | Handles data summaries, metrics, analysis        | ManagerAgent (Routing + Instruction Generator) | ManagerAgent (Routing + Instruction Generator) | Sticky Note2: Instructions to create DataAgent with input `{{json.query}}` and role description.             |
| Simple Memory2                   | Langchain Memory Buffer Window           | Memory for ManagerAgent                           | ManagerAgent (Routing + Instruction Generator) | ManagerAgent (Routing + Instruction Generator) |                                                                                                              |
| ManagerAgent (Routing + Instruction Generator) | Langchain Agent                         | Routes tasks to EmailAgent or DataAgent          | OpenAI Chat Model, Simple Memory2     | EmailAgent (Communication Specialist), DataAgent (Insight Generator) | Sticky Note: Setup instructions for ManagerAgent role and system message.                                   |
| EmailAgent (Communication Specialist) | Langchain Agent Tool                     | Writes professional emails                        | ManagerAgent (Routing + Instruction Generator) | ManagerAgent (Routing + Instruction Generator) | Sticky Note1: Setup instructions for EmailAgent with tone and input expression.                             |
| Sticky Note16                   | Sticky Note                             | Contact info for support                          | —                                    | —                                    | Contact: robert@ynteractive.com, LinkedIn: https://www.linkedin.com/in/robert-breen-29429625/                |
| Sticky Note                    | Sticky Note                             | ManagerAgent setup instructions                   | —                                    | —                                    | Step 1: Setup ManagerAgent with delegation roles and system message.                                        |
| Sticky Note1                   | Sticky Note                             | EmailAgent setup instructions                      | —                                    | —                                    | Step 4: Setup EmailAgent with professional email writing instructions and input expression.                 |
| Sticky Note2                   | Sticky Note                             | DataAgent setup instructions                       | —                                    | —                                    | Step 3: Setup DataAgent with metrics and analysis role, input `{{json.query}}`.                             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create OpenAI Credentials**  
   - Setup OpenAI API credentials with access to GPT-4o-mini model.

2. **Create the Manager Agent**  
   - Add an **Agent** node, name it `ManagerAgent (Routing + Instruction Generator)`.  
   - Set the system message:  
     ```
     You are an AI Manager that delegates tasks to specialized agents. Your job is to analyze the user's message and decide whether it requires:

     An EmailAgent for writing outreach, follow-up, or templated emails, or

     A DataAgent for tasks involving data summaries, metrics, or analysis.

     send the instructions to the sub agents.
     ```  
   - Connect an **OpenAI Chat Model** node configured with model `gpt-4o-mini` and your OpenAI credentials as the `ai_languageModel` input.  
   - Add a **Memory Buffer Window** node (`Simple Memory2`) and connect it to the manager agent’s `ai_memory` input.  
   - Connect the outputs of the manager agent to the EmailAgent and DataAgent tools using `ai_tool` connections.

3. **Create the Email Agent Tool**  
   - Add an **Agent Tool** node, name it `EmailAgent (Communication Specialist)`.  
   - In the tool description, write:  
     ```
     Writes professional, friendly, or action-oriented emails based on instructions.
     ```  
   - Use the system message:  
     ```
     You are a professional Email Writing Assistant. You write polished, effective emails for tasks such as outreach, follow-ups, and client communication. Follow the instruction provided exactly and return only the email content. Use a warm, business-appropriate tone.
     ```  
   - For the input text, use expression:  
     ```
     {{ $fromAI('Prompt__User_Message_', ``, 'string') }}
     ```  
   - Connect an **OpenAI Chat Model** node with `gpt-4o-mini` and your OpenAI credentials to this EmailAgent’s `ai_languageModel` input.  
   - Add a **Memory Buffer Window** node (`Simple Memory`) and connect it to the EmailAgent’s `ai_memory` input.  
   - Connect the EmailAgent’s `ai_tool` output back to the ManagerAgent’s `ai_tool` input.

4. **Create the Data Agent Tool**  
   - Add another **Agent Tool** node, name it `DataAgent (Insight Generator)`.  
   - Set the tool description:  
     ```
     Responds to instructions requiring metrics, summaries, or data analysis explanations.
     ```  
   - For the input text, use the expression:  
     ```
     {{json.query}}
     ```  
   - Connect an **OpenAI Chat Model** node with `gpt-4o-mini` and your OpenAI credentials to DataAgent’s `ai_languageModel` input.  
   - Add a **Memory Buffer Window** node (`Simple Memory1`) and connect it to the DataAgent’s `ai_memory` input.  
   - Connect the DataAgent’s `ai_tool` output back to the ManagerAgent’s `ai_tool` input.

5. **Final Connections**  
   - Ensure that the ManagerAgent’s `ai_tool` input receives both EmailAgent and DataAgent outputs as tools.  
   - Confirm the memory nodes are properly connected to their respective agents.  
   - Verify the OpenAI Chat Model nodes are connected as the language models for each agent.

6. **Test the Workflow**  
   - Trigger the ManagerAgent with various input queries to verify it correctly delegates to EmailAgent or DataAgent and returns coherent responses.  
   - Monitor for API errors or memory issues and adjust buffer sizes or retry settings as needed.

---

### 5. General Notes & Resources

| Note Content                                                                                           | Context or Link                                                                                         |
|------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| 📧 Contact for help or customization: robert@ynteractive.com                                         | Email support contact                                                                                   |
| 🔗 LinkedIn profile of workflow author: https://www.linkedin.com/in/robert-breen-29429625/           | Professional LinkedIn profile                                                                           |
| Step 1: Setup instructions for Manager Agent with delegation logic                                   | Sticky Note in workflow                                                                                 |
| Step 3: Setup instructions for DataAgent with input `{{json.query}}` and role description            | Sticky Note in workflow                                                                                 |
| Step 4: Setup instructions for EmailAgent with tone and input expression                             | Sticky Note in workflow                                                                                 |

---

**Disclaimer:**  
The provided text originates exclusively from an automated n8n workflow. It complies strictly with current content policies and contains no illegal, offensive, or protected elements. All processed data is legal and public.