Tesla 15min Indicators Tool (Short-Term AI Technical Analysis)

https://n8nworkflows.xyz/workflows/tesla-15min-indicators-tool--short-term-ai-technical-analysis--4096


# Tesla 15min Indicators Tool (Short-Term AI Technical Analysis)

### 1. Workflow Overview

This workflow, titled **Tesla 15min Indicators Tool (Short-Term AI Technical Analysis)**, is designed to analyze Tesla’s (TSLA) short-term market momentum and structure based on six key technical indicators on a 15-minute timeframe. It serves as an AI-powered evaluation agent within the broader **Tesla Quant Trading AI Agent** system.

**Primary Use Case:**  
Intraday traders and quant analysts use this tool to detect volatility shifts, trend strength changes, and potential reversals for Tesla stock within short intraday windows. The output is a concise structured JSON summary of market conditions feeding into a higher-level trading decision workflow.

---

**Logical Blocks:**

- **1.1 Trigger Input Reception**  
  Listens for execution triggers from the parent workflow, receiving session context and optional messages.

- **1.2 Data Acquisition via Webhook**  
  Fetches cleaned, latest 20 data points of six technical indicators for Tesla on the 15-minute timeframe via an HTTP webhook.

- **1.3 AI Reasoning and Interpretation**  
  Uses OpenAI GPT-4.1 integrated with LangChain Agent and session memory to analyze indicator data and generate a structured market summary.

- **1.4 Session Memory Management**  
  Maintains short-term memory across calls for consistent reasoning within the same session context.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger Input Reception

**Overview:**  
This block initiates the workflow when called by the parent workflow (`Tesla Financial Market Data Analyst Tool`). It accepts `message` and `sessionId` parameters to maintain session continuity and pass optional instructions.

**Nodes Involved:**  
- When Executed by Another Workflow  
- Sticky Note (Trigger Explanation)

**Node Details:**

- **When Executed by Another Workflow**  
  - Type: Execute Workflow Trigger  
  - Role: Entry point, triggered by external workflow calls  
  - Configuration: Receives `message` and `sessionId` as inputs  
  - Input: External call via Execute Workflow node  
  - Output: Passes inputs to AI Agent node  
  - Edge Cases: Missing input parameters can cause downstream logic failures; ensure parent workflow sends required data  
  - No version-specific constraints  

- **Sticky Note (Trigger Explanation)**  
  - Provides contextual information for users about input requirements  
  - No technical role  

---

#### 2.2 Data Acquisition via Webhook

**Overview:**  
This block retrieves the latest 20 data points of six Tesla technical indicators from a secured webhook endpoint, which aggregates data from the Alpha Vantage Premium API. The indicators are: RSI, MACD, BBANDS, SMA, EMA, and ADX.

**Nodes Involved:**  
- 15min Data (HTTP Request Tool)  
- Sticky Note (Data Source Explanation)

**Node Details:**

- **15min Data**  
  - Type: HTTP Request Tool  
  - Role: Fetches latest technical indicator data from webhook URL  
  - Configuration:  
    - URL: `https://treasurium.app.n8n.cloud/webhook/15minData`  
    - No additional options set (default HTTP GET)  
  - Input: None (triggered downstream from AI Agent node as an AI tool input)  
  - Output: Returns JSON data containing arrays of indicator values, limited to the last 20 records  
  - Edge Cases:  
    - Network timeouts or webhook unavailability  
    - Data format inconsistencies or missing values can disrupt AI agent parsing  
  - Version: Compatible with n8n HTTP Request Tool v4.2  

- **Sticky Note (Data Source Explanation)**  
  - Details source and data preparation: cleaned Alpha Vantage data limited to 20 points, fetched via webhook  
  - No technical role  

---

#### 2.3 AI Reasoning and Interpretation

**Overview:**  
This core block processes the indicator data using a LangChain Agent configured with OpenAI GPT-4.1. It interprets market momentum, trend strength, and volatility conditions to generate a concise JSON summary describing Tesla’s short-term trading context.

**Nodes Involved:**  
- Tesla 15min Indicators Agent (LangChain Agent)  
- OpenAI Chat Model (LM Chat OpenAI)  
- Sticky Notes (GPT Model Explanation, Tool Description)

**Node Details:**

- **Tesla 15min Indicators Agent**  
  - Type: LangChain Agent (AI Tool node)  
  - Role: Main reasoning engine; receives input message and session context, fetches indicator data, and produces structured analysis  
  - Configuration:  
    - Input text: Expression set to `{{$json.message}}` (passed from trigger)  
    - System message defines role as Tesla 15-minute indicators analyst AI, detailing indicators and expected output format  
    - Connects to `15min Data` node for indicator input, `OpenAI Chat Model` for language model, and `Simple Memory` for session context  
  - Inputs:  
    - `message` (from trigger)  
    - Indicator data (from 15min Data node)  
    - Memory context (from Simple Memory)  
  - Outputs: JSON summary describing market momentum, indicator values, and timeframe  
  - Edge Cases:  
    - Missing or malformed indicator data can cause AI response errors  
    - OpenAI API rate limits or authentication failures  
    - Expression errors if `message` input is undefined  
  - Version: Uses LangChain Agent v1.8, OpenAI GPT-4.1 model  

- **OpenAI Chat Model**  
  - Type: LangChain LM Chat OpenAI node  
  - Role: Provides GPT-4.1 language model capabilities for the agent  
  - Configuration:  
    - Model: `gpt-4.1`  
    - Credentials: OpenAI API key configured under `OpenAi account`  
  - Inputs: Receives prompt from LangChain Agent node  
  - Outputs: Textual AI reasoning passed back to Agent node  
  - Edge Cases: API key expiration, usage limits, network errors  

- **Sticky Notes (GPT Model Explanation & Tool Description)**  
  - Explain the AI model role and output format  
  - Provide context on the processing of webhook data to produce summary JSON  

---

#### 2.4 Session Memory Management

**Overview:**  
Maintains a short-term memory buffer per `sessionId` to ensure consistent reasoning across multiple calls in the same session, supporting multi-agent system coherence.

**Nodes Involved:**  
- Simple Memory (LangChain Memory Buffer Window)  
- Sticky Note (Memory Module Explanation)

**Node Details:**

- **Simple Memory**  
  - Type: LangChain Memory Buffer Window node  
  - Role: Stores conversation context based on session ID, allowing the agent to maintain history over multiple executions  
  - Configuration: Default buffer window without custom limits specified  
  - Inputs: Connected as AI memory input to the Tesla 15min Indicators Agent  
  - Outputs: Provides memory context for agent’s reasoning  
  - Edge Cases: Memory overflow if session messages are too large; session ID missing causes isolated context loss  
  - Version: v1.3  

- **Sticky Note (Memory Module Explanation)**  
  - Describes the memory node’s role in maintaining evaluation flow and session consistency  

---

### 3. Summary Table

| Node Name                      | Node Type                        | Functional Role                                | Input Node(s)                      | Output Node(s)                   | Sticky Note                                                                                  |
|-------------------------------|---------------------------------|-----------------------------------------------|----------------------------------|---------------------------------|----------------------------------------------------------------------------------------------|
| When Executed by Another Workflow | Execute Workflow Trigger         | Entry trigger receiving `message` and `sessionId` inputs | External workflow call            | Tesla 15min Indicators Agent     | Trigger from Parent Workflow: Receives sessionId and message from parent workflow           |
| Tesla 15min Indicators Agent   | LangChain Agent                 | AI reasoning core: analyzes indicators, outputs summary JSON | When Executed by Another Workflow, 15min Data, Simple Memory, OpenAI Chat Model | None (final output)              | Tesla 15-minute Technical Indicators AI: Processes webhook data and outputs JSON summary     |
| Simple Memory                  | LangChain Memory Buffer Window | Maintains session context for multi-call memory consistency | None                            | Tesla 15min Indicators Agent     | Short-Term Memory Module: Keeps session context for consistent evaluation                    |
| OpenAI Chat Model              | LangChain LM Chat OpenAI        | Provides GPT-4.1 language model for AI reasoning | Tesla 15min Indicators Agent     | Tesla 15min Indicators Agent     | GPT Model for Reasoning: Uses OpenAI GPT-4.1 to interpret technical indicators                |
| 15min Data                    | HTTP Request Tool               | Fetches latest 20 data points for 6 technical indicators from webhook | Tesla 15min Indicators Agent     | Tesla 15min Indicators Agent     | 15-Minute Indicator Data from Alpha Vantage: Fetches cleaned indicator data via webhook      |
| Sticky Note                   | Sticky Note                    | Documentation and explanation nodes           | N/A                              | N/A                             | Multiple nodes have descriptive sticky notes as per above                                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Node:**  
   - Add **Execute Workflow Trigger** node named `When Executed by Another Workflow`  
   - Configure to accept two workflow inputs: `message` (string, optional), `sessionId` (string, required)  

2. **Add HTTP Request Node for Data Fetching:**  
   - Add **HTTP Request Tool** node named `15min Data`  
   - Set URL to `https://treasurium.app.n8n.cloud/webhook/15minData`  
   - Leave method as GET and no extra options  

3. **Add LangChain Memory Node:**  
   - Add **LangChain Memory Buffer Window** node named `Simple Memory`  
   - Use default buffer settings  

4. **Add OpenAI Chat Model Node:**  
   - Add **LangChain LM Chat OpenAI** node named `OpenAI Chat Model`  
   - Select model `gpt-4.1`  
   - Assign OpenAI API credentials (create new credential if needed with your OpenAI key)  

5. **Add LangChain Agent Node:**  
   - Add **LangChain Agent** node named `Tesla 15min Indicators Agent`  
   - Set input text to expression `{{$json.message}}` to dynamically receive the message passed by trigger  
   - Paste the system message prompt specifying the agent’s role analyzing 15-minute Tesla technical indicators and expected JSON output (use the detailed system message content from the original workflow)  
   - Configure LangChain Agent to:  
     - Use the `OpenAI Chat Model` node as the language model  
     - Use `Simple Memory` node as memory input  
     - Use `15min Data` as an AI tool input to fetch indicator data  

6. **Connect Nodes:**  
   - Connect `When Executed by Another Workflow` main output to `Tesla 15min Indicators Agent` input  
   - Connect `15min Data` node output as AI tool input to `Tesla 15min Indicators Agent`  
   - Connect `Simple Memory` output as AI memory input to `Tesla 15min Indicators Agent`  
   - Connect `OpenAI Chat Model` node output to AI language model input on `Tesla 15min Indicators Agent`  

7. **Credential Setup:**  
   - Create credentials for `Alpha Vantage Premium API` as HTTP Query Auth for the webhook tool feeding `15min Data` (outside this workflow)  
   - Create OpenAI credentials with your API key and assign to `OpenAI Chat Model` node  

8. **Test Workflow Execution:**  
   - Trigger the workflow externally via `Execute Workflow` node from the parent workflow, passing a `message` string and `sessionId` to maintain memory context  
   - Validate output JSON matches expected format with summary, timeframe, and indicators object  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                              | Context or Link                                                                                                  |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| Workflow is designed as a sub-agent, **not triggered manually**; use `Execute Workflow` from the Tesla Financial Market Data Analyst Tool to invoke it with correct parameters.                                                             | Integration instruction                                                                                         |
| Requires **Tesla Quant Technical Indicators Webhooks Tool** to supply cleaned indicator data via webhook at `/15minData`.                                                                                                                | Dependent workflow link: https://n8n.io/workflows/4095-tesla-quant-technical-indicators-webhooks-tool/          |
| OpenAI GPT-4.1 powers AI reasoning; ensure API key is valid and usage limits are monitored.                                                                                                                                                 | OpenAI API usage                                                                                                 |
| The workflow output format is a strict JSON structure used downstream by the Tesla Financial Market Data Analyst Tool for final trading decisions.                                                                                        | Output format example and usage                                                                                  |
| Creator: Don Jayamaha (LinkedIn: https://linkedin.com/in/donjayamahajr), Intellectual property of Treasurium Capital Limited Company © 2025. Unauthorized use prohibited.                                                                    | Author and licensing                                                                                             |
| For detailed setup and background, see embedded sticky notes inside the workflow describing node roles and AI logic.                                                                                                                     | Internal documentation within workflow                                                                          |
| Network or API errors on the webhook or OpenAI nodes can interrupt the workflow; implement error handling or retry logic in production environment.                                                                                       | Operational considerations                                                                                       |

---

**Disclaimer:**  
The provided content is derived exclusively from an automated n8n workflow. It complies strictly with content policies and contains no illegal or protected material. All data processed is public and lawful.