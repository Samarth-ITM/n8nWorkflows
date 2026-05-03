Generate Prospect Research & Connection Strategy Reports with Claude AI

https://n8nworkflows.xyz/workflows/generate-prospect-research---connection-strategy-reports-with-claude-ai-6484


# Generate Prospect Research & Connection Strategy Reports with Claude AI

### 1. Workflow Overview

This workflow automates the generation of a personalized prospect research and connection strategy report using Claude AI via the OpenRouter interface. It is designed for sales, business development, or recruitment professionals who want to create detailed, actionable HTML reports about a prospect to improve outreach effectiveness.

The workflow includes the following logical blocks:

- **1.1 Input Reception:** Receives prospect and prospector details including names, social media URLs, and outreach goals via a trigger node designed to be called from another workflow.
- **1.2 Research Execution:** Calls an external comprehensive research sub-workflow that performs deep multi-platform research and connection analysis on the prospect and prospector.
- **1.3 AI Report Generation:** Uses Claude AI (via OpenRouter) to synthesize the research data into a structured, semantically rich HTML report with actionable insights.
- **1.4 HTML Output Preparation:** Extracts and formats the AI-generated report text into a dedicated HTML field for downstream use.
- **1.5 Documentation and Guidance:** Accompanying sticky notes provide contextual information on inputs, research methodology, and report generation.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block receives all necessary inputs including prospect and prospector identities, social media URLs, and outreach goals. It serves as the workflow’s entry point when triggered by another workflow.

- **Nodes Involved:**  
  - When Executed by Another Workflow  
  - Sticky Note (Input & User Preferences)

- **Node Details:**

  - **When Executed by Another Workflow**  
    - Type: Execute Workflow Trigger  
    - Role: Entry trigger to receive input parameters from an external workflow.  
    - Config: Expects inputs - Prospect Name, Prospect LinkedIn URL, Twitter URL, Instagram URL, Outreach Goal, Prospector Name, and corresponding social URLs.  
    - Expressions: Inputs are referenced downstream using `$('When Executed by Another Workflow').item.json['Field Name']`.  
    - Connections: Output connected to "Call Research Agent".  
    - Edge Cases: Missing or invalid URLs may reduce research quality; no explicit validation here.  
    - Version: 1.1

  - **Sticky Note (Input & User Preferences)**  
    - Type: Sticky Note  
    - Role: Documentation reminder for input correctness and importance of URLs.  
    - Content: Emphasizes the necessity of correct user inputs for accurate research.

#### 2.2 Research Execution

- **Overview:**  
  This block delegates to a separate comprehensive research agent workflow that conducts deep profile verification and multi-source research on both prospect and prospector, producing rich data for the report.

- **Nodes Involved:**  
  - Call Research Agent  
  - Sticky Note1 (Research via Multi-tool Research Agent)

- **Node Details:**

  - **Call Research Agent**  
    - Type: Execute Workflow  
    - Role: Calls sub-workflow "General Research Agent" to gather structured research data.  
    - Config: Passes a detailed research directive prompt including prospect and prospector info, social URLs, outreach goal, and comprehensive instructions for research methodology and output formatting.  
    - Uses dynamic expressions referencing inputs from the trigger node.  
    - Parameters include a randomly generated sessionId for uniqueness.  
    - Connections: Output connected to "Report Writer".  
    - Edge Cases: Sub-workflow failure, timeouts, or incomplete research output could impact report quality. Requires the sub-workflow with ID "k053fXGjIF7dUIQZ" to be available and functional.  
    - Version: 1.2

  - **Sticky Note1 (Research via Multi-tool Research Agent)**  
    - Type: Sticky Note  
    - Role: Describes the comprehensive research approach taken by the called sub-workflow and its objectives, including profile verification and connection analysis.

#### 2.3 AI Report Generation

- **Overview:**  
  Synthesizes the research data into a professional, mobile-responsive HTML report. The report includes an executive summary, background, personal interests, connection points, engagement strategy, and cited sources, all personalized for the prospector.

- **Nodes Involved:**  
  - Report Writer  
  - OpenRouter Chat Model1  
  - Sticky Note2 (Generate HTML Prospect Research Report)

- **Node Details:**

  - **OpenRouter Chat Model1**  
    - Type: Langchain OpenRouter Chat Model (Anthropic Claude Sonnet 4)  
    - Role: Provides the Claude AI language model interface to generate conversational AI output.  
    - Config: Uses model "anthropic/claude-sonnet-4" with default options.  
    - Credentials: Requires OpenRouter API credentials.  
    - Connections: Output connected to "Report Writer" as AI language model input.  
    - Edge Cases: API call failures, rate limits, or credential issues can cause errors.  
    - Version: 1

  - **Report Writer**  
    - Type: Langchain Chain LLM  
    - Role: Defines the detailed prompt instructions for Claude AI to generate the HTML report.  
    - Config: Complex, multi-point prompt detailing format, structure, citation rules, personalization, tone, and handling of insufficient data. Uses the research output (`{{ $json.output }}`) as input data.  
    - Key Expressions:  
      - Personalization references the trigger node for prospect and prospector names.  
      - Input data dynamically injected from research agent output.  
    - Connections: Output connected to "Set HTML".  
    - Edge Cases: Prompt complexity may cause generation errors or incomplete HTML. Requires retry on failure with 2500 ms delay.  
    - Version: 1.5

  - **Sticky Note2 (Generate HTML Prospect Research Report)**  
    - Type: Sticky Note  
    - Role: Explains the report’s structure, mobile-friendly HTML output, and usage scenarios such as email or posting online.

#### 2.4 HTML Output Preparation

- **Overview:**  
  Extracts the generated report text from the AI output and assigns it to a dedicated “HTML” field for subsequent consumption or delivery.

- **Nodes Involved:**  
  - Set HTML

- **Node Details:**

  - **Set HTML**  
    - Type: Set Node  
    - Role: Assigns the AI-generated report text into a variable named “HTML”.  
    - Config: Sets field "HTML" to the expression `{{ $('Report Writer').item.json.text }}`.  
    - Connections: Final node; no outputs.  
    - Edge Cases: If Report Writer output is missing or malformed, this will assign empty or invalid HTML.  
    - Version: 3.4

---

### 3. Summary Table

| Node Name                     | Node Type                                  | Functional Role                      | Input Node(s)                    | Output Node(s)             | Sticky Note                                                                                               |
|-------------------------------|--------------------------------------------|------------------------------------|---------------------------------|----------------------------|---------------------------------------------------------------------------------------------------------|
| When Executed by Another Workflow | Execute Workflow Trigger                    | Entry point, receives inputs       | -                               | Call Research Agent          | # 📝 Input & User Preferences - User inputs: Prospector name, Prospect name, and social media URLs for both. Also input the outreach goal describing why the connection is sought. These inputs feed into the research subworkflow. Make sure all URLs are valid and correct to improve research accuracy. |
| Call Research Agent             | Execute Workflow                          | Calls the research sub-workflow    | When Executed by Another Workflow | Report Writer               | # 🔍 Research via Multi-tool Research Agent - Single call to the Multi-tool Research Agent workflow. Performs profile verification, deep research, and connection analysis for both prospector and prospect. Flags uncertainties or identity verification issues. |
| OpenRouter Chat Model1          | Langchain OpenRouter Chat Model           | AI language model interface        | Report Writer (as ai_languageModel) | Report Writer              |                                                                                                         |
| Report Writer                  | Langchain Chain LLM                        | Generates HTML report from research| Call Research Agent, OpenRouter Chat Model1 | Set HTML                   | # 📄 Generate HTML Prospect Research Report - Builds a detailed, mobile-friendly HTML report addressed to the prospector with structured sections and actionable insights. |
| Set HTML                      | Set                                        | Assigns AI output text to HTML field| Report Writer                   | -                          |                                                                                                         |
| Sticky Note                   | Sticky Note                               | Documentation on inputs            | -                               | -                          | # 📝 Input & User Preferences - User inputs: Prospector name, Prospect name, and social media URLs for both. Also input the outreach goal describing why the connection is sought. These inputs feed into the research subworkflow. Make sure all URLs are valid and correct to improve research accuracy. |
| Sticky Note1                  | Sticky Note                               | Documentation on research methodology| -                             | -                          | # 🔍 Research via Multi-tool Research Agent - Single call to the Multi-tool Research Agent workflow. Performs profile verification, deep research, and connection analysis for both prospector and prospect. Flags uncertainties or identity verification issues. |
| Sticky Note2                  | Sticky Note                               | Documentation on report generation | -                               | -                          | # 📄 Generate HTML Prospect Research Report - Builds a detailed, mobile-friendly HTML report addressed to the prospector with structured sections and actionable insights. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Node:**  
   - Node Type: Execute Workflow Trigger  
   - Name: `When Executed by Another Workflow`  
   - Configure inputs: Add parameters for  
     - Prospect Name (string)  
     - Prospect Linkedin URL (string)  
     - Prospect Twitter URL (string)  
     - Prospect Instagram URL (string)  
     - Outreach Goal (string)  
     - Prospector Name (string)  
     - Prospector Linkedin URL (string)  
     - Prospector Twitter URL (string)  
     - Prospector Instagram URL (string)  

2. **Add Sticky Note for Inputs:**  
   - Content: Remind users to input valid names, URLs, and outreach goals to ensure research accuracy.  
   - Position near trigger node.

3. **Add Execute Workflow Node:**  
   - Node Type: Execute Workflow  
   - Name: `Call Research Agent`  
   - Configure:  
     - Workflow ID: Select or enter the ID of your "General Research Agent" workflow (must be available in your instance).  
     - Workflow Inputs: Provide a JSON object with the key `chatInput` containing a detailed prompt for comprehensive prospect and prospector research.  
       - Use expressions to dynamically insert fields from the trigger node, e.g., `{{ $('When Executed by Another Workflow').item.json['Prospect Name'] }}`, etc.  
       - Include a `sessionId` parameter set to a unique string, e.g., `={{ (Math.random().toString(36).substring(2) + Date.now().toString(36)) }}`.

4. **Add Sticky Note for Research Agent:**  
   - Content: Explain that this node performs deep multi-platform research, including profile verification and connection analysis.

5. **Add Langchain OpenRouter Chat Model Node:**  
   - Node Type: `@n8n/n8n-nodes-langchain.lmChatOpenRouter`  
   - Name: `OpenRouter Chat Model1`  
   - Configure:  
     - Model: `anthropic/claude-sonnet-4`  
     - Credentials: Set OpenRouter API credentials (must be preconfigured with valid API key).  

6. **Add Langchain Chain LLM Node:**  
   - Node Type: `@n8n/n8n-nodes-langchain.chainLlm`  
   - Name: `Report Writer`  
   - Configure prompt:  
     - Write a detailed prompt instructing Claude AI to generate a semantic, mobile-responsive HTML report including specific sections (Executive Summary, Background, etc.) with strict HTML formatting rules.  
     - Include personalization with expressions referencing input parameters for prospect and prospector names.  
     - Input data for the prompt should be the research output from `Call Research Agent`.  
   - Enable retry on failure with a 2500 ms wait.  

7. **Connect Nodes:**  
   - Connect `When Executed by Another Workflow` → `Call Research Agent`  
   - Connect `Call Research Agent` → `Report Writer` (as main input)  
   - Connect `OpenRouter Chat Model1` → `Report Writer` (to `ai_languageModel` input)

8. **Add Sticky Note for Report Generation:**  
   - Content: Describe the report’s HTML structure, usage scenarios, and personalization.

9. **Add Set Node:**  
   - Node Type: Set  
   - Name: `Set HTML`  
   - Configure to assign a new field `HTML` with the value from `Report Writer` node’s output text:  
     - Expression: `={{ $('Report Writer').item.json.text }}`

10. **Connect `Report Writer` → `Set HTML`.**

11. **Credential Setup:**  
    - Ensure OpenRouter API credentials are correctly configured in n8n for the Chat Model node.  
    - Ensure the "General Research Agent" workflow exists and is accessible by this workflow.

12. **Validation:**  
    - Validate all nodes for correct expressions and parameter mappings.  
    - Test with sample inputs including valid social URLs and names.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                      | Context or Link                                                                                          |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| The research sub-workflow ("General Research Agent") is critical and must be properly configured for this workflow to function. It performs multi-source data gathering and requires complex prompt handling.                      | Refer to workflow ID `k053fXGjIF7dUIQZ` or [Multi-tool Research Agent](https://n8n.io/workflows/5588-multi-tool-research-agent-for-animal-advocacy-with-openrouter-serper-and-open-paws-db/) |
| The workflow focuses on generating strictly HTML-formatted reports suitable for emails or web posting, not markdown or plain text.                                                                                              | See detailed prompt instructions in Report Writer node.                                                  |
| OpenRouter integration facilitates using Claude AI models, requiring correct API setup and stable internet connectivity.                                                                                                        | https://openrouter.ai/                                                                                   |
| Use semantic HTML5 tags and minimal inline CSS for mobile responsiveness and readability, as per prompt instructions.                                                                                                           |                                                                                                         |
| Proper handling of insufficient or conflicting data is built into the prompt instructions to ensure transparent report generation.                                                                                              |                                                                                                         |

---

This documentation allows advanced users and AI agents to fully understand, reproduce, and modify the workflow, including anticipating potential integration issues and error points.