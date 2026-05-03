Retrieve NASA Space Weather & Asteroid Data with GPT-4o-mini and Telegram

https://n8nworkflows.xyz/workflows/retrieve-nasa-space-weather---asteroid-data-with-gpt-4o-mini-and-telegram-3834


# Retrieve NASA Space Weather & Asteroid Data with GPT-4o-mini and Telegram

### 1. Workflow Overview

This workflow automates the retrieval and processing of NASA Space Weather and Asteroid data, integrating AI processing with Telegram messaging for ease of use and accessibility. It targets professionals and enthusiasts in astronomy, astrophysics, space weather, education, media, and research sectors who require an automated, no-code method to obtain, analyze, and disseminate NASA data efficiently.

The workflow is logically divided into these blocks:

- **1.1 Input Reception:** Captures user requests or commands via Telegram.
- **1.2 AI Processing Core:** Utilizes an AI Agent powered by OpenAI’s language model to interpret commands, orchestrate data retrieval, and process received data.
- **1.3 NASA Data Retrieval:** Multiple NASA nodes fetch specific space weather and asteroid datasets.
- **1.4 Output Dispatch:** Sends processed results back to the user via Telegram.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
This block listens for incoming user messages or commands on Telegram, serving as the entry point for the workflow.

- **Nodes Involved:**  
  - Start (Telegram Trigger)

- **Node Details:**  
  - **Start (Telegram Trigger):**  
    - Type: Trigger node that receives Telegram messages.  
    - Configuration: Connected to a Telegram bot credential account enabling webhook-based reception of user input.  
    - Key Expressions/Variables: Captures raw Telegram message data as input.  
    - Inputs: External Telegram user message.  
    - Outputs: Forwards message data to the AI Agent node.  
    - Edge Cases/Potential Failures: Telegram credential misconfiguration, webhook connectivity issues, message format errors.

#### 2.2 AI Processing Core

- **Overview:**  
This block interprets user input, requests data from NASA nodes, processes results with the OpenAI chat model, and prepares the output message.

- **Nodes Involved:**  
  - AI Agent  
  - OpenAI Chat Model

- **Node Details:**  
  - **AI Agent:**  
    - Type: LangChain AI Agent node acting as the workflow’s brain.  
    - Configuration: Receives Telegram input, controls data flow to NASA nodes, invokes OpenAI model for natural language understanding and response generation.  
    - Key Expressions: Uses AI language model output and NASA node data as inputs to generate processed results.  
    - Inputs: Telegram Trigger → AI Agent (main); NASA nodes → AI Agent (ai_tool); OpenAI Chat Model → AI Agent (ai_languageModel).  
    - Outputs: Sends final processed data to Telegram send node.  
    - Edge Cases: AI model timeout, malformed input, API rate limits, expression evaluation errors.  
    - Version: 1.9 (LangChain agent integration requires this or later).  
  - **OpenAI Chat Model:**  
    - Type: Language model node (OpenAI GPT-4o-mini or similar).  
    - Configuration: Connected to OpenAI credentials; used for interpreting commands and generating text responses.  
    - Inputs: AI Agent via ai_languageModel connection.  
    - Outputs: AI Agent node.  
    - Edge Cases: API key expiration, rate limiting, model selection errors.  
    - Version: 1.2.

#### 2.3 NASA Data Retrieval

- **Overview:**  
This block contains multiple NASA API nodes that fetch specific space weather and asteroid data sets. Each node communicates with NASA’s public APIs to retrieve curated data which the AI Agent then processes.

- **Nodes Involved:**  
  - Asteroid Neo-Feed  
  - Asteroid Neo-Browse  
  - DONKI High Speed Stream  
  - DONKI Interplanetary Shock  
  - DONKI Magnetopause Crossing  
  - DONKI Notification  
  - DONKI Radiation Belt Enhancement  
  - DONKI Solar Energetic Particle  
  - DONKI Solar Flare

- **Node Details:**  
  For all NASA nodes:  
  - Type: n8n’s NASA Tool node, specialized for respective NASA datasets.  
  - Configuration: Connected to NASA API credentials (single credential reused for all nodes). No additional parameters shown, implying default or dynamic query settings.  
  - Inputs: None directly from previous nodes; called asynchronously by AI Agent via the ai_tool connection.  
  - Outputs: Data passed to AI Agent for aggregation and processing.  
  - Edge Cases: API credential issues, network timeouts, empty data responses, NASA API changes or deprecations.  
  - Version: 1.

#### 2.4 Output Dispatch

- **Overview:**  
Sends processed information back to the user on Telegram as a final message.

- **Nodes Involved:**  
  - Finished (Telegram send node)

- **Node Details:**  
  - **Finished:**  
    - Type: Telegram node for sending messages.  
    - Configuration: Uses Telegram bot credentials, sends the AI Agent’s processed output text to the user.  
    - Inputs: AI Agent main output.  
    - Outputs: None (end of workflow).  
    - Edge Cases: Telegram API limits, malformed message content, credential expiration.  
    - Version: 1.2.

---

### 3. Summary Table

| Node Name                  | Node Type                         | Functional Role                 | Input Node(s)           | Output Node(s)         | Sticky Note                                                  |
|----------------------------|----------------------------------|--------------------------------|------------------------|------------------------|--------------------------------------------------------------|
| Start                      | Telegram Trigger                 | Input Reception                | —                      | AI Agent               | Connect Telegram Credential Account first.                   |
| AI Agent                   | LangChain AI Agent              | AI Processing Core             | Start, NASA nodes, OpenAI Chat Model | Finished               | Use configured AI Model and NASA nodes for data retrieval.   |
| OpenAI Chat Model           | Language Model (OpenAI)          | AI Language Model              | AI Agent                | AI Agent               | Connect OpenAI Credential; use capable model (e.g., GPT-4o-mini). |
| Asteroid Neo-Feed           | NASA Tool                       | Fetch Asteroid Data            | — (called by AI Agent) | AI Agent               | Connect NASA Credential once; used by all NASA nodes.         |
| Asteroid Neo-Browse         | NASA Tool                       | Fetch Asteroid Data            | — (called by AI Agent) | AI Agent               | Same as above.                                               |
| DONKI High Speed Stream     | NASA Tool                       | Fetch Space Weather Data       | — (called by AI Agent) | AI Agent               | Same as above.                                               |
| DONKI Interplanetary Shock  | NASA Tool                       | Fetch Space Weather Data       | — (called by AI Agent) | AI Agent               | Same as above.                                               |
| DONKI Magnetopause Crossing | NASA Tool                       | Fetch Space Weather Data       | — (called by AI Agent) | AI Agent               | Same as above.                                               |
| DONKI Notification          | NASA Tool                       | Fetch Space Weather Data       | — (called by AI Agent) | AI Agent               | Same as above.                                               |
| DONKI Radiation Belt Enhancement | NASA Tool                 | Fetch Space Weather Data       | — (called by AI Agent) | AI Agent               | Same as above.                                               |
| DONKI Solar Energetic Particle | NASA Tool                   | Fetch Space Weather Data       | — (called by AI Agent) | AI Agent               | Same as above.                                               |
| DONKI Solar Flare           | NASA Tool                       | Fetch Space Weather Data       | — (called by AI Agent) | AI Agent               | Same as above.                                               |
| Finished                   | Telegram Send Node              | Output Dispatch                | AI Agent                | —                      | Connect Telegram Credential to send results back to user.    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger node ("Start"):**  
   - Type: Telegram Trigger  
   - Set up Telegram bot credentials (OAuth2 or bot token).  
   - Configure webhook to listen for messages.  
   - Position at start of workflow.

2. **Create AI Agent node ("AI Agent"):**  
   - Type: LangChain AI Agent (v1.9 or later).  
   - Connect main input from "Start" node.  
   - Configure AI Agent to accept multiple AI tools and language model inputs.  
   - Enable outputs to the next node ("Finished").  
   - No parameters needed here as AI Agent dynamically calls other nodes.

3. **Create OpenAI Chat Model node ("OpenAI Chat Model"):**  
   - Type: LangChain OpenAI Chat Model (v1.2).  
   - Connect to AI Agent’s ai_languageModel input.  
   - Configure OpenAI credentials with your API key.  
   - Select capable model (e.g., GPT-4o-mini).  
   - No additional parameters needed unless customizing prompt or tokens.

4. **Create NASA Tool nodes:**  
   For each dataset below, create a NASA Tool node (type: n8n-nodes-base.nasaTool v1):  
   - Asteroid Neo-Feed  
   - Asteroid Neo-Browse  
   - DONKI High Speed Stream  
   - DONKI Interplanetary Shock  
   - DONKI Magnetopause Crossing  
   - DONKI Notification  
   - DONKI Radiation Belt Enhancement  
   - DONKI Solar Energetic Particle  
   - DONKI Solar Flare  
   Configure all to use the **same NASA API credential** (create or import from NASA API).  
   Connect their outputs to the AI Agent node via the ai_tool input.

5. **Create Telegram Send node ("Finished"):**  
   - Type: Telegram node (send message).  
   - Connect main input from AI Agent output.  
   - Use the same Telegram bot credentials as the "Start" node.  
   - Configure message content dynamically from AI Agent output.

6. **Connect all nodes:**  
   - Start → AI Agent (main input).  
   - NASA nodes → AI Agent (ai_tool inputs).  
   - OpenAI Chat Model → AI Agent (ai_languageModel input).  
   - AI Agent → Finished (Telegram send).  

7. **Set workflow active and save:**  
   - Verify all credentials are valid and active.  
   - Save the workflow.  
   - Activate to enable live processing.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                      | Context or Link                                                                                                    |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------|
| Connect Telegram credentials first to all Telegram nodes to enable message input and output.                                                                                     | Credential setup instructions in n8n documentation.                                                              |
| Use a single NASA API credential for all NASA Tool nodes for simplicity and easier management.                                                                                   | NASA API registration and key management at https://api.nasa.gov                                                   |
| OpenAI credential must be created and linked; recommended model is GPT-4o-mini for balanced capability and efficiency.                                                           | OpenAI API docs: https://platform.openai.com/docs                                                                 |
| Customize output formatting or additional processing by adding nodes after AI Agent as needed for specific use cases (e.g., saving to databases, forwarding to other platforms). | Workflow allows extensibility.                                                                                     |
| This workflow is designed for automation and reducing manual coding efforts in accessing NASA data and integrating AI-driven analysis.                                           | Useful for research, education, science journalism, and public science institutions.                               |
| For detailed setup and troubleshooting, refer to the official n8n guides and forums.                                                                                            | https://docs.n8n.io/ and https://community.n8n.io/                                                                 |

---

This structured documentation should empower users and AI agents to understand, reproduce, customize, and troubleshoot the workflow effectively.