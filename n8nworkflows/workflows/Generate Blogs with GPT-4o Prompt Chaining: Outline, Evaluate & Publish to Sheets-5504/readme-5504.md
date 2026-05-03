Generate Blogs with GPT-4o Prompt Chaining: Outline, Evaluate & Publish to Sheets

https://n8nworkflows.xyz/workflows/generate-blogs-with-gpt-4o-prompt-chaining--outline--evaluate---publish-to-sheets-5504


# Generate Blogs with GPT-4o Prompt Chaining: Outline, Evaluate & Publish to Sheets

### 1. Workflow Overview

This workflow automates the generation of blog posts using GPT-4o-based prompt chaining through Azure OpenAI. It targets content creators, marketers, and technical writers who want to produce high-quality, structured blog content automatically and store it in Google Sheets for publishing or tracking.

The workflow is logically grouped into the following blocks:

- **1.1 Scheduled Trigger & Topic Generation:** Automatically triggers every 5 hours to generate blog topic ideas specialized for the Sparrow API testing platform.
- **1.2 Outline Creation:** Converts the chosen blog topic into a structured blog outline with section titles and key points.
- **1.3 Outline Evaluation:** Reviews and refines the outline for engagement, clarity, logical flow, and conclusion.
- **1.4 Blog Writing:** Generates a detailed blog post from the revised outline.
- **1.5 Publishing:** Appends the generated blog post with the current date into a specified Google Sheet.

This modular approach ensures specialized AI agents handle distinct tasks, improving accuracy, debuggability, and flexibility.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Trigger & Topic Generation

- **Overview:** This block initiates the workflow at regular intervals and generates blog topic ideas tailored for the Sparrow API testing platform using an AI agent.
- **Nodes Involved:** 
  - Schedule Trigger
  - AI Agent
  - Azure OpenAI Chat Model3

- **Node Details:**

  - **Schedule Trigger**
    - *Type & Role:* Time-based trigger node.
    - *Configuration:* Fires every 5 hours (interval rule).
    - *Input/Output:* No inputs; outputs trigger signal to AI Agent.
    - *Failure Modes:* Misconfigured interval or node downtime could prevent triggering.

  - **AI Agent**
    - *Type & Role:* LangChain agent node specialized in generating blog topics.
    - *Configuration:* Text prompt instructs the agent to suggest blog topics for Sparrow API testing platform, including titles, descriptions, and user benefits.
    - *Key Expression:* Static prompt text defining the agent’s role and task.
    - *Input/Output:* Receives trigger from Schedule Trigger; outputs list of blog topics.
    - *Failure Modes:* Prompt misinterpretation, API limit exceeded, or Azure authentication issues.

  - **Azure OpenAI Chat Model3**
    - *Type & Role:* Azure OpenAI GPT-4o model node serving as the language model backend for the AI Agent.
    - *Configuration:* Uses GPT-4o with no additional options selected.
    - *Credentials:* Uses configured Azure OpenAI account.
    - *Input/Output:* Connects to AI Agent’s language model input; outputs processed text.
    - *Failure Modes:* API rate limits, credentials expired or invalid, network timeouts.

---

#### 2.2 Outline Creation

- **Overview:** Takes generated blog topics and creates a detailed outline with section titles and key points to guide blog writing.
- **Nodes Involved:** 
  - Outline Writer
  - Azure OpenAI Chat Model

- **Node Details:**

  - **Outline Writer**
    - *Type & Role:* LangChain agent for generating blog outlines.
    - *Configuration:* Prompt instructs to produce a structured blog outline with section titles and key points based on the topic.
    - *Key Expression:* `=Here is the topic to write a blog about: {{ $json.output }}` — the topic is dynamically inserted.
    - *Input/Output:* Input from AI Agent node; output is the generated outline.
    - *Failure Modes:* If topic input missing or malformed, prompt may fail; potential model errors or API issues.

  - **Azure OpenAI Chat Model**
    - *Type & Role:* GPT-4o backend powering the Outline Writer node.
    - *Configuration:* GPT-4o model on Azure OpenAI with default options.
    - *Credentials:* Azure OpenAI API credentials.
    - *Input/Output:* Connected as language model for Outline Writer.
    - *Failure Modes:* Same as other Azure OpenAI nodes.

---

#### 2.3 Outline Evaluation

- **Overview:** Reviews and refines the generated outline to ensure it meets quality criteria like engaging introduction, clear sections, logical flow, and a strong conclusion.
- **Nodes Involved:** 
  - Outline Evaluation
  - Azure OpenAI Chat Model1

- **Node Details:**

  - **Outline Evaluation**
    - *Type & Role:* LangChain agent specialized in evaluating blog outlines.
    - *Configuration:* Prompt asks the agent to revise the outline against four key criteria and output only the revised outline.
    - *Key Expression:* `=Here is the outline: \n\n{{ $json.output }}` — dynamically inputs the generated outline.
    - *Input/Output:* Input from Outline Writer; output is the refined outline.
    - *Failure Modes:* Errors if outline missing or malformed; output may be incomplete or ambiguous if prompt misunderstood.

  - **Azure OpenAI Chat Model1**
    - *Type & Role:* GPT-4o backend for Outline Evaluation.
    - *Configuration & Credentials:* Same as other GPT-4o Azure nodes.

---

#### 2.4 Blog Writing

- **Overview:** Generates a full detailed blog post using the revised outline, producing well-structured paragraphs and engaging content.
- **Nodes Involved:** 
  - Blog Writer
  - Azure OpenAI Chat Model2

- **Node Details:**

  - **Blog Writer**
    - *Type & Role:* LangChain agent for producing final blog content.
    - *Configuration:* Prompt instructs the agent to generate a detailed blog post from the provided revised outline.
    - *Key Expression:* `=Here if the revised outline: {{ $json.output }}` — receives the refined outline.
    - *Input/Output:* Input from Outline Evaluation; output is the full blog text.
    - *Failure Modes:* Possible hallucinations or verbosity if prompt not clear; API or model issues as usual.

  - **Azure OpenAI Chat Model2**
    - *Type & Role:* GPT-4o model powering Blog Writer.
    - *Configuration & Credentials:* Same as previous Azure OpenAI nodes.

---

#### 2.5 Publishing to Google Sheets

- **Overview:** Appends the generated blog post content along with the current date to a Google Sheet for storage or future use.
- **Nodes Involved:** 
  - Append row in sheet

- **Node Details:**

  - **Append row in sheet**
    - *Type & Role:* Google Sheets node for appending data.
    - *Configuration:* Appends a row with columns "Blog" (content) and "Date" (current date in dd/MM/yyyy format).
    - *Key Expressions:* 
      - Blog: `={{ $json.output }}`
      - Date: `={{ $now.format('dd/MM/yyyy') }}`
    - *Input/Output:* Input from Blog Writer; outputs appended row confirmation.
    - *Credentials:* Uses Google Sheets OAuth2 credentials.
    - *Failure Modes:* Credential expiration, network errors, sheet permissions, invalid spreadsheet ID or sheet name.

---

### 3. Summary Table

| Node Name             | Node Type                                | Functional Role                          | Input Node(s)          | Output Node(s)          | Sticky Note                                                                                                    |
|-----------------------|----------------------------------------|----------------------------------------|------------------------|-------------------------|---------------------------------------------------------------------------------------------------------------|
| Schedule Trigger       | n8n-nodes-base.scheduleTrigger          | Initiates workflow every 5 hours       | —                      | AI Agent                |                                                                                                               |
| AI Agent              | @n8n/n8n-nodes-langchain.agent          | Generates blog topic ideas              | Schedule Trigger       | Outline Writer          |                                                                                                               |
| Azure OpenAI Chat Model3 | @n8n/n8n-nodes-langchain.lmChatAzureOpenAi | Language model backend for AI Agent    | —                      | AI Agent                |                                                                                                               |
| Outline Writer         | @n8n/n8n-nodes-langchain.agent          | Creates structured blog outline         | AI Agent               | Outline Evaluation      |                                                                                                               |
| Azure OpenAI Chat Model | @n8n/n8n-nodes-langchain.lmChatAzureOpenAi | Language model backend for Outline Writer | —                      | Outline Writer          |                                                                                                               |
| Outline Evaluation     | @n8n/n8n-nodes-langchain.agent          | Evaluates and revises outline            | Outline Writer         | Blog Writer             |                                                                                                               |
| Azure OpenAI Chat Model1 | @n8n/n8n-nodes-langchain.lmChatAzureOpenAi | Language model backend for Outline Evaluation | —                      | Outline Evaluation      |                                                                                                               |
| Blog Writer            | @n8n/n8n-nodes-langchain.agent          | Generates full blog post from outline   | Outline Evaluation     | Append row in sheet     |                                                                                                               |
| Azure OpenAI Chat Model2 | @n8n/n8n-nodes-langchain.lmChatAzureOpenAi | Language model backend for Blog Writer | —                      | Blog Writer             |                                                                                                               |
| Append row in sheet    | n8n-nodes-base.googleSheets              | Appends blog and date to Google Sheet   | Blog Writer            | —                       |                                                                                                               |
| Sticky Note            | n8n-nodes-base.stickyNote                | Describes benefits of prompt chaining   | —                      | —                       | # Prompt Chaining\n ✅ Improved Accuracy and Quality – Each step focuses on a specific task, reducing errors and hallucinations.\n\n ✅ Greater Control Over Each Step – You can refine or tweak individual steps without affecting the entire process.\n\n ✅ Specialization Leads to More Effective AI Agents – Each AI agent becomes a specialist rather than a generalist, improving reliability.\n\n ✅ Easier Debugging and Optimization – If an output isn’t ideal, you only need to fix the weak link rather than redoing everything.\n\n ✅ More Scalable and Reusable Workflows – A structured, step-by-step workflow can be repurposed for different use cases.\n |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node**
   - Type: `Schedule Trigger`
   - Parameters: Set interval to trigger every 5 hours.
   - No credentials needed.

2. **Create Azure OpenAI Chat Model Node for Topic Generation**
   - Type: `LM Chat Azure OpenAI`
   - Model: `gpt-4o`
   - Credentials: Select configured Azure OpenAI API credentials.
   - Name it `Azure OpenAI Chat Model3`.

3. **Create AI Agent Node for Blog Topics**
   - Type: `LangChain Agent`
   - Prompt Type: `define`
   - Text:  
     ```
     You are an AI agent specializing in generating blog topic ideas specifically for the Sparrow API testing platform. Your role is to analyze the latest trends, user needs, and industry developments related to API testing and the Sparrow platform. When provided with information about the target audience or content objectives, you will suggest a range of blog topics tailored to Sparrow’s features, use cases, best practices, and integration tips. For each topic, include a clear title, a brief description outlining the blog’s focus, and a note on how it benefits Sparrow users or those interested in API testing. Your recommendations should be relevant, actionable, and designed to help users maximize their experience with the Sparrow API testing platform.
     ```
   - Connect `Azure OpenAI Chat Model3` as the AI language model to this node.

4. **Connect Schedule Trigger output to AI Agent input**

5. **Create Azure OpenAI Chat Model Node for Outline Writing**
   - Same settings as in step 2, name it `Azure OpenAI Chat Model`.

6. **Create Outline Writer Node**
   - Type: `LangChain Agent`
   - Prompt Type: `define`
   - Text: `=Here is the topic to write a blog about: {{ $json.output }}`
   - Options/System Message:  
     ```
     # Overview
     You are an expert outline writer. Your job is to generate a structured outline for a blog post with section titles and key points.
     ```
   - Connect `Azure OpenAI Chat Model` as AI language model.

7. **Connect AI Agent output to Outline Writer input**

8. **Create Azure OpenAI Chat Model Node for Outline Evaluation**
   - Same GPT-4o model and credentials, name it `Azure OpenAI Chat Model1`.

9. **Create Outline Evaluation Node**
   - Type: `LangChain Agent`
   - Prompt Type: `define`
   - Text: `=Here is the outline: \n\n{{ $json.output }}`
   - Options/System Message:  
     ```
     =# Overview
     You are an expert blog evaluator. Revise this outline and ensure it covers the following key criteria: 
     (1) Engaging Introduction 
     (2) Clear Section Breakdown
     (3) Logical Flow
     (4) Conclusion with Key Takeaways

     ## Output
     Only output the revised outline.
     ```
   - Connect `Azure OpenAI Chat Model1` as AI language model.

10. **Connect Outline Writer output to Outline Evaluation input**

11. **Create Azure OpenAI Chat Model Node for Blog Writing**
    - Same GPT-4o model, name it `Azure OpenAI Chat Model2`.

12. **Create Blog Writer Node**
    - Type: `LangChain Agent`
    - Prompt Type: `define`
    - Text: `=Here if the revised outline: {{ $json.output }}`
    - Options/System Message:  
      ```
      =# Overview
      You are an expert blog writer. Generate a detailed blog post using the outline with well-structured paragraphs and engaging content.
      ```
    - Connect `Azure OpenAI Chat Model2` as AI language model.

13. **Connect Outline Evaluation output to Blog Writer input**

14. **Create Google Sheets Node to Append Row**
    - Type: `Google Sheets`
    - Operation: Append
    - Sheet: Select or specify your Google Sheet by Document ID and Sheet Name (`gid=0`).
    - Columns to append:
      - `Blog`: `={{ $json.output }}`
      - `Date`: `={{ $now.format('dd/MM/yyyy') }}`
    - Credentials: Google Sheets OAuth2 credentials.

15. **Connect Blog Writer output to Append row in sheet input**

16. **Add a Sticky Note Node (Optional)**
    - Type: `Sticky Note`
    - Content:
      ```
      # Prompt Chaining
      ✅ Improved Accuracy and Quality – Each step focuses on a specific task, reducing errors and hallucinations.

      ✅ Greater Control Over Each Step – You can refine or tweak individual steps without affecting the entire process.

      ✅ Specialization Leads to More Effective AI Agents – Each AI agent becomes a specialist rather than a generalist, improving reliability.

      ✅ Easier Debugging and Optimization – If an output isn’t ideal, you only need to fix the weak link rather than redoing everything.

      ✅ More Scalable and Reusable Workflows – A structured, step-by-step workflow can be repurposed for different use cases.
      ```
    - Position it for visual aid.

---

### 5. General Notes & Resources

| Note Content                                                                                                                      | Context or Link                                                                                 |
|----------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| The workflow uses GPT-4o via Azure OpenAI, requiring valid API credentials configured within n8n.                              | Azure OpenAI official docs: https://learn.microsoft.com/en-us/azure/cognitive-services/openai/ |
| Google Sheets node requires OAuth2 credentials with permission to append rows to the specified spreadsheet.                     | Google Sheets API docs: https://developers.google.com/sheets/api                                |
| Prompt chaining improves modularity and reduces hallucinations by isolating tasks per AI agent.                                  | See sticky note content in workflow                                                           |
| The workflow is designed for Sparrow API testing platform blog content but can be adapted for other niches by modifying prompts. |                                                                                               |

---

**Disclaimer:** This documentation is based exclusively on an n8n workflow automation. It complies strictly with content policies and contains no illegal or offensive material. All data processed is legal and publicly accessible.