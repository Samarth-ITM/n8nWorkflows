Create a Complete HR Department with OpenAI O3 and GPT-4.1-mini Multi-Agent System

https://n8nworkflows.xyz/workflows/create-a-complete-hr-department-with-openai-o3-and-gpt-4-1-mini-multi-agent-system-6901


# Create a Complete HR Department with OpenAI O3 and GPT-4.1-mini Multi-Agent System

### 1. Workflow Overview

This n8n workflow, titled **"Create a Complete HR Department with OpenAI O3 and GPT-4.1-mini Multi-Agent System"**, simulates a comprehensive Human Resources (HR) department through AI-driven multi-agent collaboration. It is designed to receive HR-related chat inputs, analyze them strategically, and delegate specific HR tasks to specialized AI agents using OpenAI models. This system aims to automate and optimize HR functions such as recruitment, policy writing, training, performance management, employee engagement, and compensation analysis.

The workflow is logically divided into these main blocks:

- **1.1 Input Reception:** Captures HR-related chat messages via a webhook.
- **1.2 Strategic Analysis (CHRO Agent):** The Chief Human Resources Officer (CHRO) agent analyzes incoming requests and determines which HR specialist agents to engage.
- **1.3 Specialist Agents Execution:** Multiple specialized AI agents handle focused HR functions:
  - Recruiter Agent
  - HR Policy Writer Agent
  - Training & Development Specialist Agent
  - Performance Review Specialist Agent
  - Employee Engagement Specialist Agent
  - Compensation & Benefits Analyst Agent
- **1.4 Language Model Integration:** Each agent uses a dedicated OpenAI GPT model (O3 model for CHRO, GPT-4.1-mini for specialists) to process tasks efficiently.
- **1.5 User Guidance and Documentation:** Sticky notes provide users with comprehensive instructions, usage context, and support contacts.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
Receives chat messages that trigger the HR workflow, acting as the entry point for all incoming HR-related requests.

**Nodes Involved:**  
- When chat message received

**Node Details:**  
- **Type:** Chat Trigger (Langchain integration)  
- **Role:** Listens for incoming chat messages via webhook `hr-webhook-id`  
- **Configuration:** Basic webhook setup with no additional options. The webhook ID uniquely identifies this trigger for HR chat messages.  
- **Input/Output:** No inputs; outputs to the CHRO Agent node.  
- **Edge Cases:** Webhook unavailability, malformed messages, or unauthorized access could cause failure.  
- **Version:** 1.1  

---

#### 2.2 Strategic Analysis (CHRO Agent)

**Overview:**  
Acts as the central command, analyzing incoming HR requests and delegating tasks to appropriate specialist agents.

**Nodes Involved:**  
- CHRO Agent  
- OpenAI Chat Model CHRO  
- Think

**Node Details:**  

- **CHRO Agent**  
  - Type: Langchain Agent  
  - Role: Receives chat input, interprets strategic HR needs, and routes tasks.  
  - Config: Default options; integrates with the CHRO OpenAI model.  
  - Inputs: From "When chat message received" (chat trigger)  
  - Outputs: To all specialist agents and the Think node.  
  - Edge Cases: Misinterpretation of input, timeout, or API errors.  
  - Version: 2.1  

- **OpenAI Chat Model CHRO**  
  - Type: Langchain OpenAI Chat Model  
  - Role: Provides the cognitive engine for the CHRO agent using the "o3" OpenAI model (optimized for strategic reasoning).  
  - Config: Uses OpenAI credentials; model set explicitly to `o3` for cost-effective strategic processing.  
  - Inputs: From CHRO Agent node  
  - Outputs: Feeds CHRO Agent responses.  
  - Edge Cases: API rate limits, credential errors.  
  - Version: 1.2  

- **Think**  
  - Type: Langchain ToolThink  
  - Role: Supplements CHRO Agent by enabling deeper reasoning or internal deliberation as needed.  
  - Config: No special parameters.  
  - Inputs: From CHRO Agent  
  - Outputs: Back to CHRO Agent (closed loop)  
  - Edge Cases: Failure if the CHRO Agent does not invoke it properly.  
  - Version: 1.1  

---

#### 2.3 Specialist Agents Execution

**Overview:**  
Each specialist agent focuses on a distinct domain within HR, processing the delegated tasks using the GPT-4.1-mini model for efficiency and cost reduction.

**Nodes Involved:**  
- Recruiter Agent  
- HR Policy Writer  
- Training & Development Specialist  
- Performance Review Specialist  
- Employee Engagement Specialist  
- Compensation & Benefits Analyst  
- OpenAI Chat Models 1 to 6 (GPT-4.1-mini models mapped to each agent)

**Node Details:**  

For each specialist agent:

- **Agent Node** (e.g., Recruiter Agent)  
  - Type: Langchain Agent Tool  
  - Role: Specialized AI agent handling a specific HR function.  
  - Config: Passes the user's original message (via expression `={{ $fromAI('Prompt__User_Message_', ``, 'string') }}`) as the prompt. Contains a toolDescription specifying its domain expertise to guide the AI.  
  - Inputs: From CHRO Agent's AI tool output.  
  - Outputs: To the respective OpenAI Chat Model node.  
  - Edge Cases: Misrouting, incorrect prompt transformation, or timeouts.  
  - Version: 2.2  

- **OpenAI Chat Model Node** (e.g., OpenAI Chat Model1 for Recruiter Agent)  
  - Type: Langchain OpenAI Chat Model  
  - Role: Executes the NLP tasks with the GPT-4.1-mini model, optimized for specialist use cases to reduce cost and maintain speed.  
  - Config: Model set to `gpt-4.1-mini`, uses valid OpenAI API credentials.  
  - Inputs: From corresponding specialist agent node.  
  - Outputs: Final AI-generated response for each specialist domain.  
  - Edge Cases: API key expiry, rate limits, or malformed prompts.  
  - Version: 1.2  

**Individual Specialist Agents and their roles:**  

| Agent Name                  | Description                                                    | Model Node       |
|-----------------------------|----------------------------------------------------------------|------------------|
| Recruiter Agent             | Recruitment tasks: job descriptions, screening, interview questions | OpenAI Chat Model1 |
| HR Policy Writer            | Writing HR policies, employee handbooks, compliance docs       | OpenAI Chat Model2 |
| Training & Development Specialist | Training materials, onboarding, professional development plans | OpenAI Chat Model3 |
| Performance Review Specialist | Performance reviews, feedback, goal setting, evaluations       | OpenAI Chat Model4 |
| Employee Engagement Specialist | Employee engagement, culture initiatives, team building        | OpenAI Chat Model5 |
| Compensation & Benefits Analyst | Compensation analysis, benefits packages, salary benchmarking    | OpenAI Chat Model6 |

---

#### 2.4 User Guidance and Documentation

**Overview:**  
Provides users with clear instructions, contact info, and workflow context through sticky notes, improving usability and maintainability.

**Nodes Involved:**  
- Sticky Note Header  
- Sticky Note Main

**Node Details:**  

- **Sticky Note Header**  
  - Type: Sticky Note  
  - Role: Displays workflow title, support contact, and resource links (YouTube, LinkedIn).  
  - Content Highlights: Contact email (Yaron@nofluff.online), links to video tutorials and LinkedIn profile.  
  - Position: Top-left of the canvas for visibility.  
  - Edge Cases: None functional; purely informational.

- **Sticky Note Main**  
  - Type: Sticky Note  
  - Role: Extensive overview including workflow description, agent roles, use cases, cost optimization, and hashtags for social sharing.  
  - Content Highlights: Detailed explanations of the system architecture, use cases, and branding.  
  - Edge Cases: None functional; purely informational.

---

### 3. Summary Table

| Node Name                       | Node Type                          | Functional Role                                    | Input Node(s)                   | Output Node(s)                              | Sticky Note                                                                                         |
|--------------------------------|----------------------------------|---------------------------------------------------|--------------------------------|---------------------------------------------|---------------------------------------------------------------------------------------------------|
| When chat message received      | Langchain Chat Trigger            | Entry point receiving HR chat requests            | None                           | CHRO Agent                                  |                                                                                                   |
| CHRO Agent                     | Langchain Agent                   | Strategic HR planning and delegation               | When chat message received     | Think, Recruiter Agent, HR Policy Writer, Training & Development Specialist, Performance Review Specialist, Employee Engagement Specialist, Compensation & Benefits Analyst |                                                                                                   |
| Think                         | Langchain ToolThink               | Supports CHRO Agent's reasoning                     | CHRO Agent                    | CHRO Agent                                  |                                                                                                   |
| Recruiter Agent               | Langchain Agent Tool              | Recruitment-specific HR tasks                        | CHRO Agent                    | OpenAI Chat Model1                           |                                                                                                   |
| HR Policy Writer             | Langchain Agent Tool              | Policy writing and compliance documentation         | CHRO Agent                    | OpenAI Chat Model2                           |                                                                                                   |
| Training & Development Specialist | Langchain Agent Tool          | Training program creation and onboarding            | CHRO Agent                    | OpenAI Chat Model3                           |                                                                                                   |
| Performance Review Specialist | Langchain Agent Tool              | Performance review and evaluation                    | CHRO Agent                    | OpenAI Chat Model4                           |                                                                                                   |
| Employee Engagement Specialist | Langchain Agent Tool             | Employee engagement and culture                      | CHRO Agent                    | OpenAI Chat Model5                           |                                                                                                   |
| Compensation & Benefits Analyst | Langchain Agent Tool             | Compensation analysis and benefits                   | CHRO Agent                    | OpenAI Chat Model6                           |                                                                                                   |
| OpenAI Chat Model CHRO         | Langchain OpenAI Chat Model       | Language model powering CHRO Agent                   | CHRO Agent                    | CHRO Agent                                  |                                                                                                   |
| OpenAI Chat Model1             | Langchain OpenAI Chat Model       | GPT-4.1-mini model for Recruiter Agent               | Recruiter Agent               | None                                        |                                                                                                   |
| OpenAI Chat Model2             | Langchain OpenAI Chat Model       | GPT-4.1-mini model for HR Policy Writer              | HR Policy Writer              | None                                        |                                                                                                   |
| OpenAI Chat Model3             | Langchain OpenAI Chat Model       | GPT-4.1-mini model for Training Specialist           | Training & Development Specialist | None                                    |                                                                                                   |
| OpenAI Chat Model4             | Langchain OpenAI Chat Model       | GPT-4.1-mini model for Performance Review Specialist | Performance Review Specialist | None                                        |                                                                                                   |
| OpenAI Chat Model5             | Langchain OpenAI Chat Model       | GPT-4.1-mini model for Employee Engagement Specialist | Employee Engagement Specialist | None                                        |                                                                                                   |
| OpenAI Chat Model6             | Langchain OpenAI Chat Model       | GPT-4.1-mini model for Compensation & Benefits Analyst | Compensation & Benefits Analyst | None                                        |                                                                                                   |
| Sticky Note Header            | Sticky Note                       | Workflow title, contact info, and resource links    | None                         | None                                        | =======================================<br>CHRO AGENT WITH HR TEAM<br>Contact: Yaron@nofluff.online<br>YouTube: https://www.youtube.com/@YaronBeen/videos<br>LinkedIn: https://www.linkedin.com/in/yaronbeen/ |
| Sticky Note Main              | Sticky Note                       | Detailed overview, use cases, and branding          | None                         | None                                        | Full workflow overview with agent roles, use cases, cost optimization, and hashtags for sharing. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Chat Trigger Node**  
   - Type: `@n8n/n8n-nodes-langchain.chatTrigger`  
   - Name: "When chat message received"  
   - Set webhook ID: `hr-webhook-id`  
   - No special options.  
   - Position appropriately on canvas.

2. **Create CHRO Agent Node**  
   - Type: `@n8n/n8n-nodes-langchain.agent`  
   - Name: "CHRO Agent"  
   - Connect input to "When chat message received" main output.  
   - Leave options default.

3. **Create OpenAI Chat Model CHRO Node**  
   - Type: `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
   - Name: "OpenAI Chat Model CHRO"  
   - Set model to "o3" (OpenAI model optimized for strategic tasks).  
   - Configure OpenAI API credentials (create or link your OpenAI account).  
   - Connect output to CHRO Agent's `ai_languageModel` input.

4. **Create Think Node**  
   - Type: `@n8n/n8n-nodes-langchain.toolThink`  
   - Name: "Think"  
   - Connect CHRO Agent's `ai_tool` output to Think node input.  
   - Connect Think's output back to CHRO Agent's `ai_tool` input for reasoning support.

5. **Create Specialist Agent Nodes** (repeat for each below)  
   For each agent:  
   - Type: `@n8n/n8n-nodes-langchain.agentTool`  
   - Name accordingly (e.g., "Recruiter Agent", "HR Policy Writer", etc.)  
   - Set `text` parameter to `={{ $fromAI('Prompt__User_Message_', ``, 'string') }}` to pass user's original message.  
   - Add a descriptive `toolDescription` for the agent’s specialization.  
   - Connect the CHRO Agent’s `ai_tool` output to each specialist agent’s input.

6. **Create OpenAI Chat Model Nodes for Specialists** (1 to 6)  
   For each specialist agent:  
   - Type: `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
   - Name accordingly (e.g., "OpenAI Chat Model1" for Recruiter Agent).  
   - Set model to `gpt-4.1-mini` for cost-effective specialist processing.  
   - Configure OpenAI API credentials (use same as CHRO or separate if needed).  
   - Connect each specialist agent’s output to their corresponding OpenAI Chat Model node's input.

7. **Add Sticky Notes for Documentation**  
   - Create two sticky note nodes: "Sticky Note Header" and "Sticky Note Main".  
   - Copy content from the provided notes for user guidance and support.  
   - Position them clearly on the canvas for visibility.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                 | Context or Link                                                                                   |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| For any questions or support, contact Yaron at Yaron@nofluff.online.                                                                                                                                                                                         | Support email                                                                                    |
| Explore more tips and tutorials: YouTube channel https://www.youtube.com/@YaronBeen/videos and LinkedIn profile https://www.linkedin.com/in/yaronbeen/.                                                                                                     | Learning resources                                                                               |
| Workflow tags include #HRTech #PeopleOperations #TalentAcquisition #EmployeeExperience #HRAutomation #AIRecruitment #PerformanceManagement #CompensationBenefits #OnboardingAutomation #CultureTech #n8n #OpenAI #MultiAgentSystem #FutureOfWork #HRTransformation | Social media and discoverability tags                                                           |
| Cost optimization is achieved by using the O3 model for strategic CHRO decisions and GPT-4.1-mini models for specialist agents, reducing costs by roughly 90% while maintaining parallel processing and template caching.                                      | Design and cost efficiency insight                                                             |

---

**Disclaimer:**  
The text and workflow details provided are generated exclusively from an n8n workflow automation setup. This workflow respects all applicable content policies and does not contain illegal, offensive, or protected information. All data processed is legal and publicly accessible.