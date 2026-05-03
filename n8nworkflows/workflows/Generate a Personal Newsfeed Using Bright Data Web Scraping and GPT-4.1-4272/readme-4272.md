Generate a Personal Newsfeed Using Bright Data Web Scraping and GPT-4.1

https://n8nworkflows.xyz/workflows/generate-a-personal-newsfeed-using-bright-data-web-scraping-and-gpt-4-1-4272


# Generate a Personal Newsfeed Using Bright Data Web Scraping and GPT-4.1

### 1. Workflow Overview

This workflow generates a personalized newsfeed by leveraging Bright Data’s web scraping capabilities combined with GPT-4.1’s AI processing. It is designed to automatically collect the latest news headlines from specified sources, process and enrich the data through AI, and then send a customized newsletter email to a specified recipient.

The workflow is structured into the following logical blocks:

- **1.1 Trigger and Input Reception:** Initiates the workflow either on a scheduled daily trigger or via incoming chat messages.
- **1.2 News Collection Prompt Setup:** Prepares the input prompt instructing the AI on how and where to collect news.
- **1.3 AI Processing with Memory and Tools:** Uses GPT-4.1 with an AI Agent that integrates Bright Data’s scraping tools and memory buffer to gather and analyze news.
- **1.4 Email Dispatch:** Sends the compiled newsfeed as a newsletter email to a user-specified address.

---

### 2. Block-by-Block Analysis

#### 1.1 Trigger and Input Reception

- **Overview:**  
Activates the workflow either by a scheduled time or when a chat message is received. Provides flexible entry points for automation or manual interaction.

- **Nodes Involved:**  
  - Schedule Trigger  
  - When chat message received

- **Node Details:**

  - **Schedule Trigger**  
    - Type: Schedule Trigger  
    - Role: Automatically triggers the workflow daily at a configured hour (default 9 AM).  
    - Configuration: Set to trigger at hour 9 (can be adjusted as needed).  
    - Inputs: None  
    - Outputs: Connects to "AI news collection prompt" node.  
    - Edge Cases: Missed triggers if n8n instance is down; time zone considerations.  
    - Sticky Note: "Choose the interval that works best for you"

  - **When chat message received**  
    - Type: Langchain Chat Trigger  
    - Role: Listens for incoming chat messages to dynamically start the workflow.  
    - Configuration: Default options, webhook ID assigned for receiving chat messages.  
    - Inputs: External webhook POST requests  
    - Outputs: Connects to "AI Agent" for processing.  
    - Edge Cases: Network issues, malformed messages, authentication/authorization failures.

---

#### 1.2 News Collection Prompt Setup

- **Overview:**  
Constructs the JSON prompt that instructs the AI Agent where and how to collect news headlines using web scraping and search tools.

- **Nodes Involved:**  
  - AI news collection prompt  
  - Sticky Note (explaining purpose)

- **Node Details:**

  - **AI news collection prompt**  
    - Type: Set Node  
    - Role: Defines a JSON object with a sessionId and detailed chatInput instructing the AI to scrape specific sites and search engines for the latest news headlines.  
    - Configuration:  
      - sessionId: "google"  
      - chatInput includes URLs: https://www.brightdata.com/blog, https://www.theguardian.com/us, and Google News via the search engine tool.  
    - Inputs: Triggered by Schedule Trigger node.  
    - Outputs: Connects to "AI Agent" node.  
    - Key Expressions: Inline JSON with chatInput string directing scraping behavior.  
    - Edge Cases: Incorrect URLs or unsupported scraping targets may lead to incomplete data.

  - **Sticky Note**  
    - Content: "Injecting the AI news collection prompt for GPT-4.1"  
    - Role: Documentation of prompt injection step.

---

#### 1.3 AI Processing with Memory and Tools

- **Overview:**  
Processes the prompt using GPT-4.1, enhanced by an AI Agent that can call external web scraping tools from Bright Data, with a memory buffer to maintain session context.

- **Nodes Involved:**  
  - AI Agent  
  - OpenAI Chat Model  
  - Simple Memory  
  - List MCP Tools  
  - Scrape SERP Results  
  - Scrape Webpage  
  - Sticky Note (explaining integration)

- **Node Details:**

  - **AI Agent**  
    - Type: Langchain Agent  
    - Role: Central AI node that orchestrates the interaction between GPT-4.1, memory, and external tools.  
    - Configuration:  
      - System Message informs the agent it has access to Bright Data’s Web Unlocker API tools.  
      - Example for tool invocation is clearly given ("search_engine" exact naming required).  
    - Inputs: Receives from "AI news collection prompt" and "When chat message received".  
    - Outputs: Connects to "Send the custom newsletter via email".  
    - Retry On Fail: Enabled for robustness in case of transient errors.  
    - Edge Cases: Tool invocation failures, API rate limits, malformed system messages.  
    - Sticky Note: "Connecting to both Bright Data's MCP and your GPT model with memory"

  - **OpenAI Chat Model**  
    - Type: Langchain OpenAI Chat Model  
    - Role: Provides GPT-4.1 language model capabilities for the AI Agent.  
    - Configuration: Model set explicitly to "gpt-4.1".  
    - Inputs: Linked as AI language model input for the AI Agent.  
    - Outputs: None direct; integrated inside AI Agent.  
    - Sticky Note: "Connect using your OpenAI credentials"

  - **Simple Memory**  
    - Type: Langchain Memory Buffer Window  
    - Role: Maintains conversation or session context for the AI Agent to provide continuity.  
    - Inputs: AI Agent memory input.  
    - Outputs: Feeds back into AI Agent.  

  - **List MCP Tools**  
    - Type: Bright Data MCP Client Tool  
    - Role: Lists available scraping or data extraction tools from Bright Data.  
    - Inputs: None direct; used internally by AI Agent.  
    - Outputs: None direct; internal integration.  

  - **Scrape SERP Results**  
    - Type: Bright Data MCP Client Tool  
    - Role: Scrapes Search Engine Results Pages (SERP) for news headlines.  
    - Inputs/Outputs: Utilized by AI Agent when the "search_engine" tool is invoked.  

  - **Scrape Webpage**  
    - Type: Bright Data MCP Client Tool  
    - Role: Scrapes specific webpages for content extraction in markdown format.  
    - Inputs/Outputs: Utilized by AI Agent, e.g., for scraping Bright Data blog or Guardian news pages.

---

#### 1.4 Email Dispatch

- **Overview:**  
Sends the AI-generated personalized newsfeed as an HTML email to a configured recipient.

- **Nodes Involved:**  
  - Send the custom newsletter via email  
  - Sticky Note (explaining setup)

- **Node Details:**

  - **Send the custom newsletter via email**  
    - Type: Email Send  
    - Role: Sends the final compiled newsfeed as an email newsletter.  
    - Configuration:  
      - To Email: placeholder `<email-you-are-sending-to@something.com>` (must be replaced)  
      - From Email: placeholder `<your-from-email@something.com>` (must be replaced)  
      - Subject: "Today's Headlines"  
      - Content: HTML body from AI Agent output (`{{$json.output}}`).  
    - Inputs: Receives from AI Agent node.  
    - Outputs: None (terminal node).  
    - Edge Cases: SMTP configuration failures, email delivery issues, invalid recipient addresses.  
    - Sticky Note: "Set up with recipient and SMTP information"

---

### 3. Summary Table

| Node Name                        | Node Type                                  | Functional Role                         | Input Node(s)                | Output Node(s)                       | Sticky Note                              |
|---------------------------------|--------------------------------------------|---------------------------------------|-----------------------------|------------------------------------|-----------------------------------------|
| Schedule Trigger                | Schedule Trigger                           | Initiates workflow on schedule        | None                        | AI news collection prompt          | Choose the interval that works best for you |
| When chat message received      | Langchain Chat Trigger                     | Initiates workflow on chat message    | External webhook            | AI Agent                          |                                         |
| AI news collection prompt       | Set                                        | Constructs AI prompt JSON              | Schedule Trigger            | AI Agent                          | Injecting the AI news collection prompt for GPT-4.1 |
| AI Agent                       | Langchain Agent                            | Processes AI prompt, calls tools      | AI news collection prompt, When chat message received | Send the custom newsletter via email | Connecting to both Bright Data's MCP and your GPT model with memory |
| OpenAI Chat Model              | Langchain OpenAI Chat Model                | Provides GPT-4.1 language model       | None (AI Agent input)       | AI Agent                          | Connect using your OpenAI credentials     |
| Simple Memory                  | Langchain Memory Buffer Window             | Maintains AI conversation memory      | AI Agent                    | AI Agent                          |                                         |
| List MCP Tools                | Bright Data MCP Client Tool                 | Lists available scraping tools        | None                       | None (internal)                   |                                         |
| Scrape SERP Results           | Bright Data MCP Client Tool                 | Scrapes search engine results         | None                       | None (internal)                   |                                         |
| Scrape Webpage                | Bright Data MCP Client Tool                 | Scrapes webpage content                | None                       | None (internal)                   |                                         |
| Send the custom newsletter via email | Email Send                              | Sends personalized newsfeed email     | AI Agent                   | None                            | Set up with recipient and SMTP information |

---

### 4. Reproducing the Workflow from Scratch

1. **Create "Schedule Trigger" node**  
   - Type: Schedule Trigger  
   - Set trigger time to desired hour (default 9 AM daily).

2. **Create "When chat message received" node**  
   - Type: Langchain Chat Trigger  
   - Configure webhook to receive chat messages for manual triggering.

3. **Create "AI news collection prompt" node**  
   - Type: Set  
   - Mode: Raw JSON  
   - JSON Output:  
     ```json
     {
       "sessionId": "google",
       "chatInput": "get the latest news from https://www.brightdata.com/blog and https://www.theguardian.com/us with your scrape_as_markdown and Google News with your search engine tool to find the latest global headlines--pull actual headlines, not just the site description."
     }
     ```
   - Connect "Schedule Trigger" output to this node's input.

4. **Create "OpenAI Chat Model" node**  
   - Type: Langchain OpenAI Chat Model  
   - Set model to "gpt-4.1".  
   - Configure with valid OpenAI API credentials.

5. **Create "Simple Memory" node**  
   - Type: Langchain Memory Buffer Window  
   - Use default settings to maintain session context.

6. **Create "List MCP Tools" node**  
   - Type: Bright Data MCP Client Tool  
   - No special configuration; connects internally within AI Agent.

7. **Create "Scrape SERP Results" node**  
   - Type: Bright Data MCP Client Tool  
   - No special configuration; used by AI Agent for search engine scraping.

8. **Create "Scrape Webpage" node**  
   - Type: Bright Data MCP Client Tool  
   - No special configuration; used for scraping specific webpages.

9. **Create "AI Agent" node**  
   - Type: Langchain Agent  
   - System Message:  
     ```
     You are an expert web scraping assistant with access to Bright Data's Web Unlocker API. This gives you the ability to execute a specific set of actions. When using tools, you must share across the exact name of the tool for it to be executed.

     For example, "Search Engine Scraping" should be "search_engine"
     ```
   - Link "AI news collection prompt" and "When chat message received" outputs as inputs.  
   - Set AI language model input to "OpenAI Chat Model" node.  
   - Set AI memory input to "Simple Memory".  
   - Enable retry on failure.  
   - Connect output to "Send the custom newsletter via email".

10. **Create "Send the custom newsletter via email" node**  
    - Type: Email Send  
    - To Email: Replace with your recipient email address.  
    - From Email: Replace with your verified sender email address.  
    - Subject: "Today's Headlines"  
    - HTML Content: Use expression to pull `{{$json.output}}` from the AI Agent node.  
    - Configure SMTP credentials accordingly.

11. **Connect nodes as follows:**  
    - Schedule Trigger → AI news collection prompt  
    - AI news collection prompt → AI Agent  
    - When chat message received → AI Agent  
    - OpenAI Chat Model → AI Agent (AI language model input)  
    - Simple Memory → AI Agent (AI memory input)  
    - AI Agent → Send the custom newsletter via email

12. **Optional: Add sticky notes** to document key configuration or instructions corresponding to each logical block for user clarity.

---

### 5. General Notes & Resources

| Note Content                                                                 | Context or Link                               |
|------------------------------------------------------------------------------|-----------------------------------------------|
| Connect using your OpenAI credentials                                        | Sticky Note on "OpenAI Chat Model" node       |
| Set up with recipient and SMTP information                                  | Sticky Note on "Send the custom newsletter via email" node |
| Connecting to both Bright Data's MCP and your GPT model with memory          | Sticky Note on "AI Agent" node                  |
| Choose the interval that works best for you                                 | Sticky Note on "Schedule Trigger" node          |
| Workflow integrates Bright Data’s Web Unlocker API tools with GPT-4.1       | Core feature enabling tailored web scraping and AI processing |
| Requires valid OpenAI API credentials and SMTP server configuration          | Essential for AI generation and email dispatch  |

---

**Disclaimer:** The provided text originates exclusively from an automated workflow created with n8n, a tool for integration and automation. This processing strictly respects applicable content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.