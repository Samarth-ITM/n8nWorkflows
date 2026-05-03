Financial Analysis Report Chatbot Agent with Gemini 2.5 Flash and Discord

https://n8nworkflows.xyz/workflows/financial-analysis-report-chatbot-agent-with-gemini-2-5-flash-and-discord-7800


# Financial Analysis Report Chatbot Agent with Gemini 2.5 Flash and Discord

---

### 1. Workflow Overview

This workflow, titled **"Financial Analysis Report Chatbot Agent with Gemini and Discord"**, is designed to provide dynamic financial analysis reports through an AI chatbot interface, leveraging Google’s Gemini AI models and integrating directly with Discord for real-time communication. It targets financial analysts, investment teams, or anyone seeking on-demand financial insights in an accessible chat format.

The workflow’s core logic is organized into the following functional blocks:

- **1.1 Input Reception and Chat Interface:** Handles user input through a LangChain Chat Trigger node exposed via a public URL, enabling users to submit financial queries interactively.

- **1.2 Conversation Context Management:** Maintains conversational context with a sliding window memory buffer to provide coherent, context-aware AI responses.

- **1.3 AI Processing and Analysis:** Processes user queries via a LangChain Agent node configured with a system message tailored for financial analysis, using Google Gemini 2.5 Flash AI model for generating structured financial insights.

- **1.4 Structured Output Parsing:** Ensures the AI response adheres to a predefined JSON schema for consistency and easier downstream handling.

- **1.5 Output Cleaning and Formatting:** Cleans and formats the AI output into a Discord-friendly message, including markdown formatting and sanitization.

- **1.6 Discord Posting:** Sends the formatted report to a specified Discord channel via a webhook, including disclaimers and contextual information.

- **1.7 Documentation and Notes:** Sticky notes throughout the workflow provide setup instructions, usage guidance, customization tips, and disclaimers.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and Chat Interface

- **Overview:**  
This block captures user queries via an interactive chat interface that can be accessed publicly. It initiates the workflow execution upon receiving user input.

- **Nodes Involved:**  
  - Example Chat (LangChain Chat Trigger)

- **Node Details:**  

  - **Example Chat**  
    - *Type and Role:* LangChain Chat Trigger node; entry point for user chats.  
    - *Configuration:* Public webhook enabled; titled "Your Financial AI Agent 🚀" with a friendly initial message; input placeholder customized.  
    - *Key Expressions:* User input stored as `chatInput` JSON property.  
    - *Connections:* Outputs to the `agent1` node for processing.  
    - *Edge Cases:* Network issues could block webhook reception; malformed input could affect AI processing.  
    - *Version-Specific:* Uses version 1.1 of the node.

#### 1.2 Conversation Context Management

- **Overview:**  
Maintains recent conversation history (up to 30 messages) to provide context for the AI model, improving response coherence and relevance.

- **Nodes Involved:**  
  - Conversation Memory

- **Node Details:**  

  - **Conversation Memory**  
    - *Type and Role:* LangChain memory buffer window node; manages chat history context.  
    - *Configuration:* Sliding window length set to 30 messages by default.  
    - *Input/Output:* Receives chat input from `Example Chat`; outputs context to the `agent1` node.  
    - *Edge Cases:* Excessive context may increase costs or latency; resetting memory needed during testing.  
    - *Version-Specific:* Version 1.3.

#### 1.3 AI Processing and Analysis

- **Overview:**  
The core AI agent analyzes the user's financial queries using a specialized system prompt, generating structured markdown insights based on the Gemini 2.5 Flash Lite model.

- **Nodes Involved:**  
  - Connect Gemini  
  - agent1

- **Node Details:**  

  - **Connect Gemini**  
    - *Type and Role:* LangChain LM Chat node for Google Gemini AI; interfaces with Gemini API.  
    - *Configuration:* Uses model "gemini-2.5-flash-lite"; temperature set to 0 for deterministic output; max tokens 2048.  
    - *Credentials:* Uses Google Palm API key securely stored in n8n credentials.  
    - *Input/Output:* Receives prompt from `agent1`; outputs AI response back to `agent1`.  
    - *Edge Cases:* API key errors, rate limits, timeouts, or model availability issues.  
    - *Version-Specific:* Version 1.

  - **agent1**  
    - *Type and Role:* LangChain Agent node; manages prompt construction, AI interaction, and output parsing.  
    - *Configuration:* Contains a detailed system message instructing the AI to analyze financial queries strictly from a financial perspective with a conversational and frank tone. Output is requested in markdown format.  
    - *Input/Output:* Receives chat input and context memory; sends prompt to `Connect Gemini`; receives AI response; outputs parsed result to the next node.  
    - *Edge Cases:* Parsing errors, prompt formatting issues, or unexpected AI output.  
    - *Version-Specific:* Version 2.2; uses output parser integration (Structured Output Parser).  
    - *Notes:* System message can be customized for tone and detail.

#### 1.4 Structured Output Parsing

- **Overview:**  
Validates and enforces the structure of the AI’s JSON response to prevent downstream errors and facilitate consistent formatting.

- **Nodes Involved:**  
  - Structured Output Parser

- **Node Details:**  

  - **Structured Output Parser**  
    - *Type and Role:* LangChain output parser node; validates AI output against a JSON schema.  
    - *Configuration:* Schema expects keys like `idea` (thesis) and `analysis` (markdown content); auto-fix enabled to correct minor JSON issues.  
    - *Input/Output:* Processes AI output from `agent1`; outputs structured JSON for formatting.  
    - *Edge Cases:* AI output deviating from schema; auto-fix may not resolve complex errors causing retries.  
    - *Version-Specific:* Version 1.3.

#### 1.5 Output Cleaning and Formatting

- **Overview:**  
Sanitizes the markdown analysis output by removing special characters unsuitable for Discord, then maps relevant fields for the Discord message.

- **Nodes Involved:**  
  - Code  
  - Edit Fields

- **Node Details:**  

  - **Code**  
    - *Type and Role:* Code node running JavaScript; cleans the AI-generated markdown by removing all special characters except letters, numbers, and spaces.  
    - *Configuration:* Uses a regex replacement to sanitize `output.analysis`.  
    - *Input/Output:* Input from `agent1`; output to `Edit Fields`.  
    - *Edge Cases:* Over-sanitization may remove meaningful markdown formatting; consider adjusting regex if needed.  
    - *Version-Specific:* Version 2.

  - **Edit Fields**  
    - *Type and Role:* Set node; maps and renames fields for Discord message consumption.  
    - *Configuration:* Assigns `clean` text, AI’s `idea` output, original chat input, and workflow name into JSON fields for Discord.  
    - *Input/Output:* Input from `Code`; output to `Discord` node.  
    - *Edge Cases:* Field mismatches with Discord template may cause display issues.  
    - *Version-Specific:* Version 3.4.

#### 1.6 Discord Posting

- **Overview:**  
Sends the final formatted financial analysis report to a specified Discord channel using a webhook, including disclaimers and user query context.

- **Nodes Involved:**  
  - Discord

- **Node Details:**  

  - **Discord**  
    - *Type and Role:* Native Discord node; posts messages to a Discord channel via webhook.  
    - *Configuration:* Message content template includes workflow name, user query, AI-generated executive summary and analysis, plus a detailed disclaimer about the nature of AI-generated financial information.  
    - *Credentials:* Uses Discord Webhook API credential securely stored in n8n.  
    - *Input/Output:* Receives formatted data from `Edit Fields`.  
    - *Edge Cases:* Webhook misconfiguration, Discord rate limits, or message length restrictions (~2000 characters).  
    - *Version-Specific:* Version 2.

#### 1.7 Documentation and Notes

- **Overview:**  
Sticky notes provide comprehensive instructions, setup guidance, customization tips, disclaimers, and references for users and maintainers.

- **Nodes Involved:**  
  - Multiple sticky notes scattered across the workflow (Introduction Note, Sticky Note12, Sticky Note13, etc.)

- **Node Details:**  

  - These nodes are informational only, containing:  
    - Setup instructions for Gemini API and Discord webhook creation.  
    - Chat trigger usage guide.  
    - Agent customization and prompt control advice.  
    - Memory management and structured output parsing tips.  
    - Disclaimer about the non-advisory nature of the AI outputs.  
    - Useful external links to Google AI Studio, Discord webhook documentation, and n8n credential setup.

---

### 3. Summary Table

| Node Name           | Node Type                              | Functional Role                          | Input Node(s)        | Output Node(s)       | Sticky Note                                                                                      |
|---------------------|--------------------------------------|----------------------------------------|----------------------|----------------------|-------------------------------------------------------------------------------------------------|
| Introduction Note    | Sticky Note                          | Setup overview and instructions        | —                    | —                    | © 2025 Aditya Vadaganadam                                                                        |
| Sticky Note12       | Sticky Note                          | Chat trigger guidance                   | —                    | —                    | © 2025 Aditya Vadaganadam                                                                        |
| Sticky Note13       | Sticky Note                          | Agent instructions and customization   | —                    | —                    | © 2025 Aditya Vadaganadam                                                                        |
| Example Chat        | LangChain Chat Trigger               | User input reception                    | —                    | agent1               | Chat trigger; public URL; demo message                                                          |
| Conversation Memory | LangChain Memory Buffer Window       | Maintains chat context                  | Example Chat          | agent1               | Keeps recent 30 messages for AI context                                                        |
| Connect Gemini      | LangChain LM Chat (Google Gemini)    | AI model interaction                    | agent1                | agent1               | Uses Google Gemini 2.5 Flash Lite model; API key required                                        |
| agent1              | LangChain Agent                     | AI financial analysis and response     | Example Chat, Memory, Connect Gemini, Structured Output Parser | Code                 | Financial analysis AI agent with custom system message                                         |
| Structured Output Parser | LangChain Output Parser Structured | Validates AI JSON output                | agent1                | agent1               | Enforces output schema; auto-fixes minor JSON issues                                            |
| Code                | Code Node                           | Cleans AI markdown output               | agent1                | Edit Fields           | Removes special characters from analysis                                                       |
| Edit Fields         | Set Node                           | Maps fields for Discord message         | Code                  | Discord               | Assigns cleaned output, idea, user input, workflow name                                        |
| Discord             | Discord Node                       | Posts message to Discord channel        | Edit Fields            | —                    | Posts report with disclaimers; uses Discord webhook credential                                  |
| Sticky Note         | Sticky Note                          | Gemini API key setup instructions      | —                    | —                    | Guide to create and use Google API key                                                         |
| Sticky Note1        | Sticky Note                          | Discord webhook setup instructions     | —                    | —                    | Official Discord docs and n8n Discord credential docs                                          |
| Sticky Note2        | Sticky Note                          | Workflow overview summary               | —                    | —                    | Describes purpose and ideal use cases                                                          |
| Sticky Note3        | Sticky Note                          | Customization guidance                   | —                    | —                    | Details prompt control, output format, scheduling, and data enrichment                          |
| Sticky Note15       | Sticky Note                          | Conversation memory explanation         | —                    | —                    | Explains context window and memory reset                                                       |
| Sticky Note16       | Sticky Note                          | Structured output parser explanation    | —                    | —                    | Describes JSON schema and validation                                                          |
| Sticky Note17       | Sticky Note                          | Discord message formatting tips         | —                    | —                    | Formatting considerations and Discord message limits                                          |
| Sticky Note18       | Sticky Note                          | Edit Fields mapping explanation         | —                    | —                    | Guidance on field selection and naming for Discord message                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Chat Trigger Node**  
   - Add a LangChain Chat Trigger node named **Example Chat**.  
   - Configure it as public with title "Your Financial AI Agent 🚀".  
   - Set the initial message to "Hi there! 👋" and input placeholder "Type your message here...".  
   - Ensure webhook is enabled for public access.

2. **Add Conversation Memory Node**  
   - Add a LangChain Memory Buffer Window node named **Conversation Memory**.  
   - Set context window length to 30 messages.  
   - Connect output of **Example Chat** to this node.

3. **Add Google Gemini LM Chat Node**  
   - Add a LangChain LM Chat Google Gemini node named **Connect Gemini**.  
   - Select model "models/gemini-2.5-flash-lite".  
   - Set temperature to 0 and max output tokens to 2048.  
   - Create or select a Google Palm API credential with your Gemini API key.  
   - Assign this credential to the node.

4. **Add LangChain Agent Node**  
   - Add a LangChain Agent node named **agent1**.  
   - Configure the System Message with the financial analysis instructions, specifying tone (conversational, sarcastic, educational), output format (markdown), and expected content (risks, drivers, takeaways).  
   - Enable output parser integration.  
   - Connect **Example Chat** (user input), **Conversation Memory** (context), and **Connect Gemini** (AI model) in the agent’s input parameters.

5. **Add Structured Output Parser Node**  
   - Add a LangChain Output Parser Structured node named **Structured Output Parser**.  
   - Define a JSON schema requiring keys like `idea` (string) and `analysis` (string markdown).  
   - Enable "Auto-fix" to correct minor JSON errors.  
   - Connect output from **agent1** to this node.

6. **Add Code Node for Output Cleaning**  
   - Add a Code node named **Code**.  
   - Insert JavaScript code to remove special characters from the `analysis` field except letters, numbers, and spaces.  
   - Input from **Structured Output Parser**; output to **Edit Fields**.

7. **Add Edit Fields Node**  
   - Add a Set node named **Edit Fields**.  
   - Map the cleaned text as `clean`, AI’s `idea` as `output.idea`, user chat input as `chatInput`, and the workflow name as `$workflow.name`.  
   - Connect from **Code** node; output to **Discord** node.

8. **Add Discord Node**  
   - Add a Discord node named **Discord**.  
   - Configure message content template combining workflow name, user query, AI summary, detailed analysis, and a full disclaimer.  
   - Create or select Discord Webhook API credentials with your Discord webhook URL.  
   - Connect input from **Edit Fields**.

9. **Link Nodes**  
   - Connect nodes in the following order:  
     `Example Chat` → `Conversation Memory` → `agent1` → `Structured Output Parser` → `Code` → `Edit Fields` → `Discord`  
   - Connect **agent1** to **Connect Gemini** for AI processing.

10. **Add Sticky Notes (Optional but Recommended)**  
    - Create sticky note nodes with setup instructions for Gemini API keys, Discord webhook creation, usage guidance, customization options, disclaimers, and best practices.  
    - Position them logically near related nodes.

11. **Set Workflow Settings**  
    - Enable public webhook access for the chat trigger.  
    - Ensure all credentials (Google Palm API and Discord Webhook) are configured and tested.  
    - Optionally set workflow to active and test with example inputs (e.g., ticker: AAPL, timeframe: 6M, question: "Key drivers and risks?").

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                | Context or Link                                                                                              |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------|
| Official Discord Docs on creating and managing webhooks                                                                                                     | https://support.discord.com/hc/en-us/articles/228383668-Intro-to-Webhooks                                    |
| n8n Docs on Discord credentials including webhook setup                                                                                                     | https://docs.n8n.io/integrations/builtin/credentials/discord/                                               |
| Google AI Studio API key creation guide                                                                                                                     | https://aistudio.google.com/app/apikey                                                                      |
| Workflow author and copyright notice: © 2025 Aditya Vadaganadam                                                                                             | Workflow metadata                                                                                           |
| Disclaimer: AI-generated financial insights are for informational purposes only and not professional advice. Users should verify independently.             | Included in Discord message content                                                                         |
| Suggested input variables: ticker, timeframe, question, risk_profile, output_format, channel_override                                                        | Customization notes in sticky notes                                                                         |
| Suggested AI models: Gemini 1.5 Pro for reasoning depth, Gemini 1.5 Flash for speed; current workflow uses Gemini 2.5 Flash Lite                            | Customizable in `Connect Gemini` node                                                                       |
| Conversation memory improves AI coherence but increases token usage; adjust `contextWindowLength` accordingly                                               | Sticky Note15                                                                                                |
| Structured output parser ensures consistent JSON output, preventing downstream formatting failures                                                          | Sticky Note16                                                                                                |
| Discord message length limits (~2000 chars) require concise formatting or message splitting                                                                 | Sticky Note17                                                                                                |

---

**Disclaimer:** The provided text and workflow relate solely to a legal and publicly available automation created using n8n. It adheres strictly to content policies and contains no illegal or protected materials.

---