Market Research & Business Case Generator with GPT-4o, Perplexity & Claude

https://n8nworkflows.xyz/workflows/market-research---business-case-generator-with-gpt-4o--perplexity---claude-5430


# Market Research & Business Case Generator with GPT-4o, Perplexity & Claude

### 1. Workflow Overview

This workflow automates the generation of a comprehensive market research and business case study using advanced AI models and live research tools. It is designed to take a user’s chat prompt describing a market opportunity or business idea, define a structured research scope, perform deep web research, synthesize findings into a formal business case, and save the final output into a Google Docs document.

**Target Use Cases:**  
- Market research and analysis  
- Business opportunity evaluation  
- Strategic planning and decision support  
- Automated content generation for consultants, analysts, entrepreneurs, and product managers

**Logical Blocks:**  
- **1.1 Input Reception:** Captures user chat input via a Langchain chat trigger node.  
- **1.2 Research Scope Definition:** Uses OpenAI GPT-4o to analyze the input and break it into structured research components.  
- **1.3 Deep Research:** Performs live, real-time data research using Perplexity Sonar based on the defined scope.  
- **1.4 Business Case Writing:** Synthesizes research data into a well-organized, professional business case study using Anthropic Claude Sonnet.  
- **1.5 Output Document Generation:** Inserts the generated business case text into a specified Google Docs document for easy access and sharing.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
Captures user input as chat messages to trigger the workflow start.

- **Nodes Involved:**  
  - When chat message received  
  - Sticky Note (Chat Input Trigger)

- **Node Details:**  

  **When chat message received**  
  - Type: Langchain Chat Trigger  
  - Role: Entry point capturing user chat input; triggers workflow on new message  
  - Configuration: Default chat trigger with webhook enabled for external input  
  - Input: None (trigger node)  
  - Output: Passes chat input JSON data downstream  
  - Edge Cases: Webhook connectivity issues, malformed chat payloads  
  - Version-specific: Requires Langchain support v1.1 or newer  

  **Sticky Note (Chat Input Trigger)**  
  - Type: Annotation node for clarity  
  - Content: “Chat Input Trigger” to label the starting point

---

#### 2.2 Research Scope Definition

- **Overview:**  
Analyzes the user’s query to define a structured market research scope, breaking it down into industry, geography, trends, opportunities, and data sources. Prepares a prompt for the Perplexity research node.

- **Nodes Involved:**  
  - Research Scope Definer Agent (OpenAI GPT-4o)  
  - Sticky Note1 (Define Research Scope)

- **Node Details:**  

  **Research Scope Definer Agent**  
  - Type: OpenAI GPT Chat Model node  
  - Role: Processes raw user input to create a detailed research prompt  
  - Configuration: Uses “chatgpt-4o-latest” model with system instructions defining a market research planner persona and output format  
  - Key Expressions: Inputs `$json.chatInput` (user message), outputs structured prompt based on listed components  
  - Input: Chat input JSON from trigger  
  - Output: Structured prompt for Perplexity research  
  - Edge Cases: API rate limits, malformed input, unexpected user queries, possible incomplete scope generation  
  - Credentials: Requires OpenAI API credentials  
  - Version: v1.8  

  **Sticky Note1 (Define Research Scope)**  
  - Annotation describing the block’s function  

---

#### 2.3 Deep Research

- **Overview:**  
Executes real-time deep web research using Perplexity Sonar model, guided by the structured prompt, returning well-cited, segmented market research data.

- **Nodes Involved:**  
  - Perplexity Business Case Deep Research  
  - Sticky Note2 (Perplexity Deep Research)

- **Node Details:**  

  **Perplexity Business Case Deep Research**  
  - Type: Perplexity node for live research  
  - Role: Retrieves and summarizes up-to-date market data based on the AI-generated scope  
  - Configuration: Uses “sonar-deep-research” model with a system prompt instructing detailed, cited sections on trends, consumers, competition, regulations, and data  
  - Key Expressions: Inputs `$json.message.content` from the GPT-4o scope definer output  
  - Input: Structured research prompt  
  - Output: Research summary with markdown citations  
  - Edge Cases: Connectivity or API failures, incomplete data retrieval, rate limits  
  - Credentials: Requires Perplexity API credentials  
  - Version: v1  

  **Sticky Note2 (Perplexity Deep Research)**  
  - Annotation describing this research phase  

---

#### 2.4 Business Case Writing

- **Overview:**  
Synthesizes the research findings into a formal, 1500-word business case study with defined sections, using Anthropic Claude Sonnet.

- **Nodes Involved:**  
  - Claude Business Case Writer (Anthropic LLM Chain)  
  - Anthropic Chat Model  
  - Sticky Note3 (Business Case Builder)

- **Node Details:**  

  **Claude Business Case Writer**  
  - Type: Langchain ChainLLM node using Anthropic model  
  - Role: Generates the final comprehensive business case study text based on Perplexity research  
  - Configuration: Uses “claude-sonnet-4-20250514” model; prompt instructs 7 structured sections with formal tone, excluding internal “thinking” content from Perplexity’s output  
  - Key Expressions: Reads Perplexity output `$json.choices[0].message.content`  
  - Input: Research summary text  
  - Output: Finished business case text  
  - Edge Cases: Token length limits, generation errors, model downtime  
  - Credentials: Anthropic API credentials required  
  - Version: v1.7  

  **Anthropic Chat Model**  
  - Connected internally as the language model backend for the chain node  

  **Sticky Note3 (Business Case Builder)**  
  - Annotation for this synthesis phase  

---

#### 2.5 Output Document Generation

- **Overview:**  
Updates a specified Google Docs document by inserting the generated business case study text.

- **Nodes Involved:**  
  - Google Docs  
  - Sticky Note4 (Business Case Output on Google Docs)

- **Node Details:**  

  **Google Docs**  
  - Type: Google Docs node  
  - Role: Inserts the generated business case text into a target Google Docs file  
  - Configuration: Uses “update” operation with “insert” action, inserting the case study text at the end or specified location  
  - Key Expressions: Text content mapped from Claude Business Case Writer output `$json.text`  
  - Input: Business case text  
  - Output: Confirmation of update action  
  - Edge Cases: Credential expiration, permission errors, invalid document URL, network issues  
  - Credentials: Requires Google Docs OAuth2 credentials with write access  
  - Version: v2  

  **Sticky Note4 (Business Case Output on Google Docs)**  
  - Annotation describing output destination  

---

### 3. Summary Table

| Node Name                        | Node Type                          | Functional Role                              | Input Node(s)                      | Output Node(s)                   | Sticky Note                                                                                                                        |
|---------------------------------|----------------------------------|----------------------------------------------|----------------------------------|---------------------------------|------------------------------------------------------------------------------------------------------------------------------------|
| When chat message received       | Langchain Chat Trigger            | Entry point for user chat input               | None                             | Research Scope Definer Agent     | Chat Input Trigger                                                                                                                 |
| Sticky Note                     | Sticky Note                      | Annotation for chat input trigger             | None                             | None                            | Chat Input Trigger                                                                                                                 |
| Research Scope Definer Agent     | OpenAI GPT Chat Model             | Defines structured research scope             | When chat message received       | Perplexity Business Case Deep Research | Define Research Scope                                                                                                              |
| Sticky Note1                    | Sticky Note                      | Annotation for research scope definition      | None                             | None                            | Define Research Scope                                                                                                              |
| Perplexity Business Case Deep Research | Perplexity Sonar Node           | Performs live deep web research                | Research Scope Definer Agent     | Claude Business Case Writer      | Perplexity Deep Research                                                                                                          |
| Sticky Note2                    | Sticky Note                      | Annotation for deep research                   | None                             | None                            | Perplexity Deep Research                                                                                                          |
| Claude Business Case Writer      | Anthropic LLM Chain              | Synthesizes research into business case       | Perplexity Business Case Deep Research | Google Docs                     | Business Case Builder                                                                                                             |
| Anthropic Chat Model             | Anthropic Chat Model             | Underlying LLM for business case generation   | Claude Business Case Writer      | None                            | Business Case Builder                                                                                                             |
| Sticky Note3                    | Sticky Note                      | Annotation for business case writing           | None                             | None                            | Business Case Builder                                                                                                             |
| Google Docs                     | Google Docs                     | Inserts final business case into Google Doc   | Claude Business Case Writer      | None                            | Business Case Output on Google Docs                                                                                              |
| Sticky Note4                    | Sticky Note                      | Annotation for output document                  | None                             | None                            | Business Case Output on Google Docs                                                                                              |
| Sticky Note5                    | Sticky Note                      | Project overview, instructions, and credits   | None                             | None                            | 🧠 Market Research Case Study Generator… For more build + step-by-step video tutorials, check out: https://www.youtube.com/@Automatewithmarc |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Chat Trigger Node:**  
   - Add a Langchain Chat Trigger node named “When chat message received”  
   - Configure with default webhook settings to receive chat messages externally  
   - No special parameters needed  

2. **Add Sticky Note for Input:**  
   - Insert a Sticky Note near the trigger node with content “Chat Input Trigger” for documentation  

3. **Add Research Scope Definer Agent Node:**  
   - Add an OpenAI GPT Chat Model node named “Research Scope Definer Agent”  
   - Set model to “chatgpt-4o-latest” (ensure OpenAI credentials are set)  
   - Configure messages as follows:  
     - User message content: `{{$json.chatInput}}`  
     - System prompt: instruct as market research planner and Perplexity prompt generator, breaking down input into Industry, Geography, Trends, Opportunities, Data Sources  
   - Connect output of “When chat message received” to this node  

4. **Add Sticky Note for Scope Definition:**  
   - Add Sticky Note with “Define Research Scope” near the Research Scope node  

5. **Add Perplexity Sonar Node for Deep Research:**  
   - Add a Perplexity node named “Perplexity Business Case Deep Research”  
   - Select model “sonar-deep-research”  
   - Set system prompt to instruct professional market analyst using reliable data sources, outputting sections with markdown citations (include Industry trends, Consumer behavior, Competitive landscape, Regulatory factors, Key data)  
   - Bind input message to output of Research Scope Definer Agent’s prompt (`{{$json.message.content}}`)  
   - Ensure Perplexity API credentials are configured  
   - Connect Research Scope Definer Agent output to this node  

6. **Add Sticky Note for Deep Research:**  
   - Add Sticky Note “Perplexity Deep Research” near Perplexity node  

7. **Add Anthropic Chain LLM Node to Write Business Case:**  
   - Add a Langchain ChainLLM node named “Claude Business Case Writer”  
   - Select Anthropic model “claude-sonnet-4-20250514” (set Anthropic API credentials)  
   - Configure the prompt to receive Perplexity output text and generate a 1500-word business case with sections: Executive Summary, Market Overview, Opportunity Analysis, Competitive Landscape, Challenges and Risks, Strategic Recommendations, Conclusion  
   - Use formal, analytical tone; exclude Perplexity “thinking” step content  
   - Connect output of Perplexity node to this node  

8. **Add Anthropic Chat Model Node:**  
   - Add a separate Anthropic Chat Model node named “Anthropic Chat Model”  
   - Set to same Claude model as used above  
   - Connect this node as the language model backend for the ChainLLM node (depending on n8n version, this may be automatic or manual)  

9. **Add Sticky Note for Business Case Builder:**  
   - Add Sticky Note “Business Case Builder” near the Claude nodes  

10. **Add Google Docs Node:**  
    - Add a Google Docs node named “Google Docs”  
    - Set operation to “Update” and action to “Insert” text  
    - Provide the target Google Docs document URL (ensure OAuth2 Google credentials with write access)  
    - Map the input text to the output business case text from Claude node (`{{$json.text}}`)  
    - Connect output of Claude Business Case Writer to Google Docs node  

11. **Add Sticky Note for Output Document:**  
    - Add Sticky Note “Business Case Output on Google Docs” near Google Docs node  

12. **Add Project Overview Sticky Note:**  
    - Optionally add a large Sticky Note with full project description, usage instructions, and credits (content from Sticky Note5) for documentation  

13. **Test the Workflow:**  
    - Deploy and activate the workflow  
    - Send a chat prompt (e.g., “Give me a market opportunity analysis of a bicycle rental business in North Africa.”) via the Langchain chat interface  
    - Verify the Google Docs document updates with a structured business case study  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                           | Context or Link                                                        |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------|
| 🧠 Market Research Case Study Generator automates comprehensive business case creation using AI + live research. Ideal for consultants, analysts, entrepreneurs, and educators.           | Project description in Sticky Note5                                   |
| For more builds and step-by-step video tutorials, visit: https://www.youtube.com/@Automatewithmarc                                                                                       | YouTube channel for further learning                                  |
| The workflow uses three cutting-edge AI models: OpenAI GPT-4o for planning, Perplexity Sonar for live research, and Anthropic Claude Sonnet for synthesis.                                | Important to have credentials for all three APIs                      |
| The Google Docs output allows easy sharing and collaboration of generated business cases.                                                                                                | Requires Google Docs API OAuth2 credentials with appropriate scopes   |
| Ensure API usage limits and credentials are monitored to avoid workflow failures due to rate limiting or authorization errors.                                                           | General operational note                                              |

---

**Disclaimer:**  
The text provided originates exclusively from an automated workflow created in n8n, an integration and automation tool. This processing strictly respects prevailing content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly available.