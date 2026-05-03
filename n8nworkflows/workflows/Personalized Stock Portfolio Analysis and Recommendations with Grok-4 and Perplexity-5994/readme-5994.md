Personalized Stock Portfolio Analysis and Recommendations with Grok-4 and Perplexity

https://n8nworkflows.xyz/workflows/personalized-stock-portfolio-analysis-and-recommendations-with-grok-4-and-perplexity-5994


# Personalized Stock Portfolio Analysis and Recommendations with Grok-4 and Perplexity

### 1. Workflow Overview

This workflow, titled **"Personalized Stock Portfolio Analysis and Recommendations with Grok-4 and Perplexity,"** automates daily stock portfolio analysis by integrating AI-powered insights with real-time market news and personalized investment recommendations. It targets retail investors, portfolio managers, fintech startups, and anyone seeking automated, data-driven stock updates without manually sifting through news articles.

The workflow is structured into the following logical blocks:

- **1.1 Scheduled Trigger:** Initiates the workflow daily at 10:00 AM to ensure timely updates.
- **1.2 AI Stock Analyst Agent:** Core AI logic that ingests current portfolio data, searches latest stock market news, analyzes the impact, and generates buy/sell recommendations.
- **1.3 Data Sources Integration:** Connects to Google Sheets for portfolio data and Perplexity for recent news search.
- **1.4 AI Language Models and Memory:** Uses xAI Grok-4 models for natural language understanding and generation, with session memory buffering.
- **1.5 Summary Generation:** Converts complex AI output into an easy-to-read email summary.
- **1.6 Email Delivery:** Sends the final summary via Gmail to a specified recipient.
- **1.7 Documentation and Notes:** Sticky notes for explanations and references.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Trigger

- **Overview:**  
  Starts the workflow execution daily at 10:00 AM, ensuring consistent timing for portfolio updates.

- **Nodes Involved:**  
  - `Schedule Trigger`  
  - `Sticky Note` (annotating the trigger)

- **Node Details:**  
  - **Schedule Trigger**  
    - Type: `n8n-nodes-base.scheduleTrigger`  
    - Configuration: Triggers once daily at 10:00 AM (hour-based interval).  
    - Inputs: None (start node).  
    - Outputs: Connects to `Grok-4 Stock Analyst Agent`.  
    - Potential Failures: None typical, unless n8n scheduler service is down.  
    - Version: 1.2  
  - **Sticky Note**  
    - Provides a visual label "On Schedule Trigger" to clarify purpose.

---

#### 2.2 AI Stock Analyst Agent

- **Overview:**  
  This block drives the core AI logic. It commands the AI agent to:  
  1. Use Google Sheets data on current stock holdings.  
  2. Query Perplexity for stock market news within the last 24 hours.  
  3. Analyze impact and provide buy/sell recommendations.

- **Nodes Involved:**  
  - `Grok-4 Stock Analyst Agent`  
  - `Stock Holdings Portfolio` (Google Sheets)  
  - `Perplexity` (news search)  
  - `Simple Memory` (session memory)  
  - `xAI Grok Chat Model` (Grok-4 language model)  
  - `Sticky Note1` (label for the agent)

- **Node Details:**  
  - **Grok-4 Stock Analyst Agent**  
    - Type: `@n8n/n8n-nodes-langchain.agent`  
    - Role: Orchestrates AI instructions combining portfolio data and news.  
    - Configuration: Static prompt defining instructions for the agent, e.g., use Google Sheets, Perplexity, provide insights.  
    - Inputs:  
      - AI tool inputs from Google Sheets and Perplexity nodes.  
      - AI language model from `xAI Grok Chat Model`.  
      - AI memory from `Simple Memory`.  
    - Outputs: Connects to `Summary Agent`.  
    - Potential Failures:  
      - Auth errors if Google Sheets or Perplexity credentials are invalid.  
      - Timeout or API rate limits with Perplexity or xAI.  
      - Expression errors if prompt or input data is malformed.  
    - Version: 2  
  - **Stock Holdings Portfolio**  
    - Type: `n8n-nodes-base.googleSheetsTool`  
    - Role: Reads current stock holdings from a designated Google Sheet (sheet URL and name configured).  
    - Inputs: None (data source).  
    - Outputs: Feeds into AI agent's tool input.  
    - Credentials: Google Sheets OAuth2 required.  
    - Version: 4.6  
    - Edge Cases:  
      - Sheet ID or name misconfigured leads to failure.  
      - OAuth token expiry or permission issues.  
  - **Perplexity**  
    - Type: `n8n-nodes-base.perplexityTool`  
    - Role: Searches stock market news within last 24 hours.  
    - Inputs: Message content dynamically set from AI agent output (overridden by AI).  
    - Outputs: Connected as AI tool input to the agent.  
    - Credentials: Perplexity API key required.  
    - Edge Cases: API rate limits, invalid API keys, or no news found.  
  - **Simple Memory**  
    - Type: `@n8n/n8n-nodes-langchain.memoryBufferWindow`  
    - Role: Maintains session context for AI agent to improve continuity.  
    - Configuration: Uses workflow ID as session key, custom session ID type.  
    - Inputs: Connected to AI agent's memory interface.  
    - Outputs: Connected back to AI agent.  
    - Edge Cases: Memory overflow or data mismatch could cause contextual errors.  
    - Version: 1.3  
  - **xAI Grok Chat Model**  
    - Type: `@n8n/n8n-nodes-langchain.lmChatXAiGrok`  
    - Role: Provides the natural language processing power (Grok-4 model).  
    - Configuration: Model set to `grok-4-0709`.  
    - Inputs: Connected as AI language model to the agent.  
    - Credentials: xAI API key required.  
    - Edge Cases: API limits, invalid credentials, service downtime.  
    - Version: 1

---

#### 2.3 Summary Generation

- **Overview:**  
  Converts the AI stock analyst agent’s detailed output into a concise, human-readable email summary.

- **Nodes Involved:**  
  - `Summary Agent`  
  - `xAI Grok Chat Model1`  
  - `Sticky Note2` (label for summary agent)

- **Node Details:**  
  - **Summary Agent**  
    - Type: `@n8n/n8n-nodes-langchain.agent`  
    - Role: Processes analyst output JSON text and formats it as an email-friendly summary.  
    - Configuration: System message defines role as a helpful summary assistant. Input text is dynamically set from previous node output.  
    - Inputs: Main input from `Grok-4 Stock Analyst Agent`.  
    - Outputs: Connects to `Gmail`.  
    - Version: 2  
    - Edge Cases: Failure if previous output is empty or malformed.  
  - **xAI Grok Chat Model1**  
    - Type: `@n8n/n8n-nodes-langchain.lmChatXAiGrok`  
    - Role: Language model for the summary agent (Grok-4).  
    - Configuration: Same model `grok-4-0709`.  
    - Inputs: AI language model input for `Summary Agent`.  
    - Credentials: xAI API key required.  
    - Edge Cases: Same as prior xAI node.

---

#### 2.4 Email Delivery

- **Overview:**  
  Sends the formatted summary to a configured email address using Gmail.

- **Nodes Involved:**  
  - `Gmail`  
  - `Sticky Note3` (label for email output)

- **Node Details:**  
  - **Gmail**  
    - Type: `n8n-nodes-base.gmail`  
    - Role: Sends an email with the summary content as plain text.  
    - Configuration:  
      - Recipient email set to `youremail@example.com` (placeholder).  
      - Subject: “Stock Updates.”  
      - Message: Dynamically set from previous node’s output JSON property `output`.  
    - Credentials: Requires Gmail OAuth2 (configured with valid OAuth2 credentials).  
    - Inputs: From `Summary Agent`.  
    - Edge Cases:  
      - Invalid email address.  
      - OAuth token expired or revoked.  
      - Gmail API quota exceeded.  
    - Version: 2.1

---

#### 2.5 Documentation and Notes

- **Overview:**  
  Sticky notes provide explanations, usage context, branding, and external resources.

- **Nodes Involved:**  
  - `Sticky Note` (Schedule Trigger)  
  - `Sticky Note1` (Grok-4 Stock Analyst Agent)  
  - `Sticky Note2` (Summary Agent)  
  - `Sticky Note3` (Email Output)  
  - `Sticky Note4` (Workflow description, use cases, and external links)

- **Key Content Highlights:**  
  - Workflow runs daily at 10 AM by default.  
  - Integrates Grok-4 AI with Perplexity and Google Sheets for personalized market insights.  
  - Provides buy/sell/hold recommendations and email summaries.  
  - Use cases include retail investors, portfolio managers, fintech startups.  
  - YouTube tutorial link: https://youtu.be/OXzsh-Ba-8Y  
  - Google Sheet template link: https://docs.google.com/spreadsheets/d/1074dZk-vhwz6LML5zoiwHdxg89Z8u_mgl7wwzqf3A98/edit?usp=sharing

---

### 3. Summary Table

| Node Name                 | Node Type                              | Functional Role                      | Input Node(s)                  | Output Node(s)              | Sticky Note                                                                                       |
|---------------------------|--------------------------------------|-----------------------------------|-------------------------------|-----------------------------|-------------------------------------------------------------------------------------------------|
| Schedule Trigger          | n8n-nodes-base.scheduleTrigger       | Start daily execution              | None                          | Grok-4 Stock Analyst Agent   | On Schedule Trigger                                                                              |
| Grok-4 Stock Analyst Agent | @n8n/n8n-nodes-langchain.agent       | AI analysis orchestration          | Schedule Trigger, Stock Holdings Portfolio, Perplexity, Simple Memory, xAI Grok Chat Model | Summary Agent                | Grok-4 Stock Analyst Agent                                                                      |
| Stock Holdings Portfolio  | n8n-nodes-base.googleSheetsTool      | Fetch portfolio data from Sheets   | None                          | Grok-4 Stock Analyst Agent   |                                                                                                 |
| Perplexity                | n8n-nodes-base.perplexityTool        | Fetch recent stock market news     | Grok-4 Stock Analyst Agent (AI tool) | Grok-4 Stock Analyst Agent   |                                                                                                 |
| Simple Memory             | @n8n/n8n-nodes-langchain.memoryBufferWindow | AI session memory buffering        | Grok-4 Stock Analyst Agent (AI memory) | Grok-4 Stock Analyst Agent   |                                                                                                 |
| xAI Grok Chat Model       | @n8n/n8n-nodes-langchain.lmChatXAiGrok | Language model for stock analyst   | Grok-4 Stock Analyst Agent (AI languageModel) | Grok-4 Stock Analyst Agent   |                                                                                                 |
| Summary Agent             | @n8n/n8n-nodes-langchain.agent       | Summarizes AI analysis output      | Grok-4 Stock Analyst Agent     | Gmail                       | Grok-4 Summary Agent                                                                            |
| xAI Grok Chat Model1      | @n8n/n8n-nodes-langchain.lmChatXAiGrok | Language model for summary agent   | Summary Agent (AI languageModel) | Summary Agent               |                                                                                                 |
| Gmail                    | n8n-nodes-base.gmail                  | Sends summary email                | Summary Agent                 | None                        | Email Output                                                                                     |
| Sticky Note              | n8n-nodes-base.stickyNote             | Annotation                        | None                          | None                        | On Schedule Trigger                                                                              |
| Sticky Note1             | n8n-nodes-base.stickyNote             | Annotation                        | None                          | None                        | Grok-4 Stock Analyst Agent                                                                      |
| Sticky Note2             | n8n-nodes-base.stickyNote             | Annotation                        | None                          | None                        | Grok-4 Summary Agent                                                                            |
| Sticky Note3             | n8n-nodes-base.stickyNote             | Annotation                        | None                          | None                        | Email Output                                                                                     |
| Sticky Note4             | n8n-nodes-base.stickyNote             | Workflow description & links      | None                          | None                        | 📊 Grok-4 with Perplexity Stock Investment Analyst – Personalized Daily Market Insights See https://youtu.be/OXzsh-Ba-8Y |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a `Schedule Trigger` node:**  
   - Set to trigger once daily at 10:00 AM (hour = 10).  
   - This node will start the workflow.

3. **Add a `Google Sheets` node (named `Stock Holdings Portfolio`):**  
   - Select the Google Sheets tool node.  
   - Configure with your Google Sheets OAuth2 credentials.  
   - Set the target Google Sheet document ID (your portfolio sheet).  
   - Set the Sheet name to the relevant tab (e.g., "Sheet1").  
   - This node fetches current stock holdings.

4. **Add a `Perplexity` node:**  
   - Configure with your Perplexity API credentials.  
   - Set search recency to "day" to get news from the last 24 hours.  
   - Configure the message content to be dynamically set by the AI agent in the workflow.

5. **Add an `xAI Grok Chat Model` node:**  
   - Select the `lmChatXAiGrok` node type.  
   - Set the model to `grok-4-0709`.  
   - Add your xAI API credentials.

6. **Add a `Simple Memory` node:**  
   - Use the `memoryBufferWindow` node type.  
   - Configure `sessionKey` as the current workflow ID (expression `{{$workflow.id}}`).  
   - Set `sessionIdType` to `customKey`.

7. **Add a `LangChain Agent` node (named `Grok-4 Stock Analyst Agent`):**  
   - Set prompt type to `define`.  
   - Provide the following prompt text:  
     ```
     As a stock analyst agent:
     1. Please use the Google Sheet tool titled "Stock Holdings Portfolio" for my current stock holdings information
     2. Please research the latest stock market news in the last 24 hours using the Perplexity tool, and tell me how the latest stock market news affected my portfolio
     3. Give some analyst insights + buy/sell recommendations based on the news
     ```
   - Connect AI tools inputs:  
     - Google Sheets node output  
     - Perplexity node output  
   - Connect AI language model input from the `xAI Grok Chat Model`.  
   - Connect AI memory input from `Simple Memory`.  
   - Connect main output to the Summary Agent (next step).

8. **Add another `xAI Grok Chat Model` node (`xAI Grok Chat Model1`):**  
   - Same configuration as the first Grok node (`grok-4-0709`, xAI API credentials).  
   - This will be used by the summary agent.

9. **Add a `LangChain Agent` node (`Summary Agent`):**  
   - Set prompt type to `define`.  
   - Input the system message:  
     ```
     You are a helpful summary assistant, summarize the information received from user and output easily readable email format of the summary.
     ```  
   - Set the input text dynamically from the `Grok-4 Stock Analyst Agent` output (expression `$json.output`).  
   - Connect AI language model input from `xAI Grok Chat Model1`.  
   - Connect main output to the Gmail node.

10. **Add a `Gmail` node:**  
    - Configure with Gmail OAuth2 credentials.  
    - Set recipient email to your desired address.  
    - Subject: "Stock Updates"  
    - Message body: Set dynamically from the summary agent output (`$json.output`).  
    - Connect input from `Summary Agent`.

11. **Connect the nodes as follows:**  
    - `Schedule Trigger` → `Grok-4 Stock Analyst Agent`  
    - `Stock Holdings Portfolio` → `Grok-4 Stock Analyst Agent` (as AI tool input)  
    - `Perplexity` → `Grok-4 Stock Analyst Agent` (as AI tool input)  
    - `Simple Memory` ↔ `Grok-4 Stock Analyst Agent` (memory input/output)  
    - `xAI Grok Chat Model` → `Grok-4 Stock Analyst Agent` (language model input)  
    - `Grok-4 Stock Analyst Agent` → `Summary Agent`  
    - `xAI Grok Chat Model1` → `Summary Agent` (language model input)  
    - `Summary Agent` → `Gmail`

12. **Add sticky notes for clarity if desired:**  
    - Label each major block (Schedule Trigger, Grok-4 Stock Analyst Agent, Summary Agent, Email Output).  
    - Include a detailed sticky note describing the workflow purpose and external resources.

13. **Test the workflow:**  
    - Ensure all credentials are valid and authorized.  
    - Run manually or wait for scheduled execution.  
    - Verify email receipt with expected summary content.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  | Context or Link                                                                                   |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| 📊 Grok-4 with Perplexity Stock Investment Analyst – Personalized Daily Market Insights. This workflow acts as a personal AI stock analyst powered by Grok-4, Perplexity, and Google Sheets to give daily tailored market insights based on your actual investment portfolio. Runs daily at 10 AM by default and delivers buy/sell/hold recommendations via email. Ideal for retail investors, portfolio managers, fintech startups.                                                                                                                                                                                 | Sticky Note4 content in workflow.                                                                |
| To watch the step-by-step build of this workflow, check out the YouTube video: https://youtu.be/OXzsh-Ba-8Y                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                | https://youtu.be/OXzsh-Ba-8Y                                                                     |
| Google Sheet template for portfolio input: https://docs.google.com/spreadsheets/d/1074dZk-vhwz6LML5zoiwHdxg89Z8u_mgl7wwzqf3A98/edit?usp=sharing                                                                                                                                                                                                                                                                                                                                                                                                                                                                             | Google Sheets template link                                                                       |

---

**Disclaimer:**  
The text provided is exclusively derived from an automated workflow created using n8n, a no-code automation tool. All content complies with applicable policies and contains no illegal or offensive material. All handled data is legal and public.