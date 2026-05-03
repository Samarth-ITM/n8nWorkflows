Track Brand Mentions with Sentiment Analysis using Bright Data & OpenAI

https://n8nworkflows.xyz/workflows/track-brand-mentions-with-sentiment-analysis-using-bright-data---openai-5951


# Track Brand Mentions with Sentiment Analysis using Bright Data & OpenAI

---

### 1. Workflow Overview

This workflow automates the process of tracking brand mentions—in this case, OpenAI—within Medium blog posts by scraping content, analyzing sentiment, and saving the results for monitoring and research purposes. It is designed for marketers, researchers, founders, or analysts who want to monitor brand presence and sentiment in blog content without manual reading or coding.

The workflow is logically divided into three core blocks:

- **1.1 Start and Input Reception:** Manual trigger to initiate the workflow and input the Medium blog URL to analyze.
- **1.2 AI-Powered Content Scraping and Sentiment Analysis:** An AI Agent orchestrates web scraping via Bright Data MCP tools and processes the scraped content using OpenAI models to extract structured insights, including sentiment.
- **1.3 Result Storage:** The processed, structured data is appended to a Google Sheet for archival and further analysis.

---

### 2. Block-by-Block Analysis

#### 2.1 Start and Input Reception

- **Overview:**  
  This block triggers the workflow manually and captures the URL of the Medium blog post to be analyzed. It serves as the entry point for user interaction.

- **Nodes Involved:**  
  - 🚀 Start Workflow (Manual Trigger)  
  - 📝 Define Medium Blog URL

- **Node Details:**

  1. **🚀 Start Workflow (Manual Trigger)**  
     - Type: Manual Trigger  
     - Role: Initiates the workflow when executed manually by the user.  
     - Configuration: No parameters needed; simple manual start.  
     - Inputs: None (start node)  
     - Outputs: Connected to "📝 Define Medium Blog URL" node.  
     - Edge Cases: User may forget to input a valid URL; no validation here.  
     - Notes: This node is renamed for clarity.

  2. **📝 Define Medium Blog URL**  
     - Type: Set  
     - Role: Collects and sets the Medium blog URL as a workflow variable for downstream nodes.  
     - Configuration: Defines a single string variable `blogURL` with a default/example Medium blog URL.  
     - Inputs: From manual trigger node.  
     - Outputs: Passes the URL to the AI agent node.  
     - Expressions: The URL is set as a static string but can be modified at runtime.  
     - Edge Cases: If URL is malformed or not a Medium URL, downstream scraping may fail.  
     - Notes: Provides a simple way for non-technical users to input the URL.

#### 2.2 AI-Powered Content Scraping and Sentiment Analysis

- **Overview:**  
  This core block uses an AI Agent to scrape the Medium blog post via Bright Data MCP tools, then processes the content with OpenAI models to extract mentions of OpenAI, analyze sentiment, and structure the data.

- **Nodes Involved:**  
  - 🤖 Agent: Scrape Medium Blog (OpenAI Mentions)  
  - 🧠 Chat Reasoning Engine (OpenAI GPT-4.1-mini)  
  - 🌐 Bright Data Tool (Bright Data MCP Client)  
  - OpenAI Chat Model (gpt-4.1-mini)  
  - Structured Output Parser  
  - Auto-fixing Output Parser

- **Node Details:**

  1. **🤖 Agent: Scrape Medium Blog (OpenAI Mentions)**  
     - Type: Langchain Agent Node  
     - Role: Orchestrates scraping and AI processing. Receives the URL and commands the scraping tool and language models to analyze content for OpenAI mentions and sentiment.  
     - Configuration:  
       - Prompt includes the blog URL and instructs scraping and sentiment analysis.  
       - Uses internal prompt type “define” with output parser enabled.  
     - Inputs: Receives `blogURL` from the Set node.  
     - Outputs: Passes structured results to Google Sheets node.  
     - Edge Cases: Network or scraping tool failures, invalid URL, API rate limits.  
     - Sub-Workflow: Coordinates sub-tasks using connected nodes (Bright Data tool, OpenAI models, output parsers).

  2. **🧠 Chat Reasoning Engine**  
     - Type: Langchain OpenAI Chat Model (gpt-4.1-mini)  
     - Role: Provides natural language understanding and reasoning capabilities for the agent to analyze scraped content.  
     - Configuration: Uses GPT-4.1-mini model with default options.  
     - Credentials: Requires valid OpenAI API key.  
     - Inputs: Connected internally to the Agent node for language processing needs.  
     - Outputs: Feeds reasoning results back to the Agent.  
     - Edge Cases: API quota exhaustion, response timeouts, model updates affecting output.

  3. **🌐 Bright Data Tool**  
     - Type: MCP Client Tool (Bright Data)  
     - Role: Executes the "scrape_as_markdown" tool to visit the Medium blog URL and scrape content in markdown format.  
     - Configuration: Tool parameters are dynamically provided by AI instructions.  
     - Credentials: Requires Bright Data MCP client API credentials.  
     - Inputs: Triggered by the Agent node.  
     - Outputs: Provides raw scraped content to the AI Agent.  
     - Edge Cases: Scraping block by target site, network errors, tool misconfiguration.

  4. **OpenAI Chat Model**  
     - Type: Langchain OpenAI Chat Model (gpt-4.1-mini)  
     - Role: Processes specific goals like filtering for OpenAI mentions and summarizing content.  
     - Configuration: Same model version; used for content understanding within the agent.  
     - Credentials: Uses same OpenAI API credentials as Chat Reasoning Engine.  
     - Inputs & Outputs: Connected in the agent workflow for enhanced content processing.

  5. **Structured Output Parser**  
     - Type: Langchain Output Parser (Structured)  
     - Role: Converts AI-generated textual output into a structured JSON format with fields such as platform, author, title, content summary, and sentiment.  
     - Configuration: Uses a JSON schema example for consistent output structure.  
     - Inputs: Receives AI output from the OpenAI Chat Model.  
     - Outputs: Passes structured JSON to the Auto-fixing Output Parser node.  
     - Edge Cases: Parsing failures due to malformed AI output or unexpected format.

  6. **Auto-fixing Output Parser**  
     - Type: Langchain Output Parser (Autofixing)  
     - Role: Automatically retries parsing if the structured output parser fails, providing error diagnostics and prompting for corrected output.  
     - Configuration: Includes instructions to retry only valid formatted answers.  
     - Inputs: Connected from Structured Output Parser and OpenAI Chat Model outputs.  
     - Outputs: Returns fixed structured output back to the Agent node.  
     - Edge Cases: Persistent parsing errors, infinite retry loops if AI fails to comply.

#### 2.3 Result Storage

- **Overview:**  
  This block appends the analyzed and structured data into a Google Sheet for storage, reporting, and historical tracking.

- **Nodes Involved:**  
  - 📥 Save Results to Google Sheets

- **Node Details:**

  1. **📥 Save Results to Google Sheets**  
     - Type: Google Sheets Node  
     - Role: Appends a new row to a specified Google Sheet containing URL, title, author, platform, sentiment, and content summary.  
     - Configuration:  
       - Mapping is explicitly defined for each column.  
       - Target sheet is identified by document ID and sheet name (`gid=0`).  
       - Operation set to "append" to avoid overwriting.  
     - Credentials: Requires Google Sheets OAuth2 authentication.  
     - Inputs: Receives structured output JSON from the Agent node.  
     - Outputs: None (terminal node).  
     - Edge Cases: API quota limits, permission errors, invalid sheet ID, data type mismatches.

---

### 3. Summary Table

| Node Name                             | Node Type                      | Functional Role                     | Input Node(s)                       | Output Node(s)                         | Sticky Note                                                                                       |
|-------------------------------------|--------------------------------|-----------------------------------|-----------------------------------|---------------------------------------|-------------------------------------------------------------------------------------------------|
| 🚀 Start Workflow (Manual Trigger)  | Manual Trigger                 | Entry point, manual start          | None                              | 📝 Define Medium Blog URL              | Start workflow manually; initiate with blog URL input                                           |
| 📝 Define Medium Blog URL            | Set                           | Input URL definition               | 🚀 Start Workflow                  | 🤖 Agent: Scrape Medium Blog (OpenAI Mentions) | Allows user to input Medium blog URL for scraping                                               |
| 🤖 Agent: Scrape Medium Blog (OpenAI Mentions) | Langchain Agent Node           | Orchestrates scraping & AI analysis | 📝 Define Medium Blog URL          | 📥 Save Results to Google Sheets       | AI Agent scrapes blog and analyzes mentions/sentiment                                           |
| 🧠 Chat Reasoning Engine             | Langchain OpenAI Chat Model   | Provides reasoning for scraping   | Connected internally in Agent     | Connected internally in Agent          | AI reasoning engine to understand and process content                                           |
| 🌐 Bright Data Tool                  | MCP Client Tool (Bright Data) | Executes scraping on Medium URL   | Connected internally in Agent     | Connected internally in Agent          | Uses Bright Data to scrape content as markdown                                                  |
| OpenAI Chat Model                   | Langchain OpenAI Chat Model   | Processes content for mentions    | Connected internally in Agent     | Connected internally in Agent          | Processes extracted content to find relevant mentions and prepare summaries                     |
| Structured Output Parser             | Langchain Output Parser       | Formats AI output into JSON       | OpenAI Chat Model                 | Auto-fixing Output Parser              | Parses AI text output into structured JSON format                                               |
| Auto-fixing Output Parser            | Langchain Output Parser       | Auto-corrects parsing errors      | Structured Output Parser, OpenAI Chat Model | 🤖 Agent: Scrape Medium Blog (OpenAI Mentions) | Retries parsing on failure to ensure valid structured output                                    |
| 📥 Save Results to Google Sheets     | Google Sheets Node            | Stores results in Google Sheets   | 🤖 Agent: Scrape Medium Blog       | None                                 | Saves blog metadata, sentiment, and summaries for tracking                                     |
| Sticky Note                         | Sticky Note                   | Documentation                    | None                              | None                                 | Section 1: Start the Workflow & Provide Input                                                  |
| Sticky Note1                        | Sticky Note                   | Documentation                    | None                              | None                                 | Section 2: AI Agent Fetches & Filters Content                                                  |
| Sticky Note3                        | Sticky Note                   | Documentation                    | None                              | None                                 | Section 3: Store the Results in Google Sheets                                                  |
| Sticky Note2                        | Sticky Note                   | Documentation                    | None                              | None                                 | Full workflow description and section summaries                                               |
| Sticky Note9                        | Sticky Note                   | Contact & support info           | None                              | None                                 | Workflow assistance and external links                                                        |
| Sticky Note5                        | Sticky Note                   | Affiliate link                  | None                              | None                                 | Bright Data referral link                                                                     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger Node**  
   - Type: Manual Trigger  
   - No special configuration  
   - Name it: "🚀 Start Workflow (Manual Trigger)"

2. **Add a Set Node to Define Medium Blog URL**  
   - Type: Set  
   - Add a string variable named `blogURL`  
   - Set a default value like `https://medium.com/gitconnected/why-openai-suddenly-erased-jony-ive-from-their-website-5d6f431e5297`  
   - Name it: "📝 Define Medium Blog URL"  
   - Connect this node from the Manual Trigger node.

3. **Add a Langchain Agent Node for Scraping and Analysis**  
   - Type: `@n8n/n8n-nodes-langchain.agent`  
   - Parameters:  
     - Prompt text: `scrape the below medium blog URL and do setimant analysis:\n{{ $json.blogURL }}`  
     - Enable output parser  
   - Name it: "🤖 Agent: Scrape Medium Blog (OpenAI Mentions)"  
   - Connect this node from "📝 Define Medium Blog URL"

4. **Within or connected to the Agent node, configure the following sub-nodes:**

   - **Chat Reasoning Engine Node**  
     - Type: `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
     - Model: `gpt-4.1-mini`  
     - Credentials: Configure with valid OpenAI API key  
     - Name: "🧠 Chat Reasoning Engine"  
     - Link this node as language model used by the Agent.

   - **Bright Data MCP Client Tool Node**  
     - Type: `n8n-nodes-mcp.mcpClientTool`  
     - Tool Name: `scrape_as_markdown`  
     - Operation: `executeTool`  
     - Tool Parameters: Accept dynamic input from AI prompt  
     - Credentials: Add Bright Data MCP client API credentials  
     - Name: "🌐 Bright Data Tool"  
     - Link this node as AI tool used by the Agent.

   - **OpenAI Chat Model Node**  
     - Type: `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
     - Model: `gpt-4.1-mini`  
     - Credentials: Same OpenAI API key as above  
     - Name: "OpenAI Chat Model"  
     - Used internally by the Agent for content processing.

   - **Structured Output Parser Node**  
     - Type: `@n8n/n8n-nodes-langchain.outputParserStructured`  
     - Provide JSON schema example with expected fields: platform, author, title, content summary, sentiment  
     - Name: "Structured Output Parser"  
     - Connect output of OpenAI Chat Model.

   - **Auto-fixing Output Parser Node**  
     - Type: `@n8n/n8n-nodes-langchain.outputParserAutofixing`  
     - Configure prompt to retry with instructions and error diagnostics  
     - Name: "Auto-fixing Output Parser"  
     - Connect output of Structured Output Parser and OpenAI Chat Model  
     - Connect output back to Agent node as corrected structured output.

5. **Add a Google Sheets Node to Store Results**  
   - Type: Google Sheets  
   - Operation: Append row  
   - Map columns explicitly:  
     - URL → `={{ $('📝 Define Medium Blog URL').item.json.blogURL }}`  
     - Title → `={{ $json.output.title }}`  
     - Author → `={{ $json.output.author }}`  
     - Platform → `={{ $json.output.platform }}`  
     - Sentiment → `={{ $json.output.sentiment }}`  
     - Content Summary → `={{ $json.output['content summary'] }}`  
   - Configure Sheet Name and Document ID accordingly (e.g., your target Google Sheet)  
   - Credentials: Google Sheets OAuth2  
   - Name: "📥 Save Results to Google Sheets"  
   - Connect this node from the Agent node.

6. **Connect the workflow nodes in this order:**  
   Manual Trigger → Set Medium Blog URL → AI Agent (with its subnodes) → Google Sheets.

7. **Additional Setup:**  
   - Ensure OpenAI API credentials are properly configured with required scopes and quota.  
   - Configure Bright Data MCP client API credentials with access to scraping tools.  
   - Ensure Google Sheets API OAuth2 credentials have edit permissions on the target spreadsheet.

---

### 5. General Notes & Resources

| Note Content                                                                                                      | Context or Link                                                                                      |
|------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| Workflow author contact for support: Yaron@nofluff.online                                                       | Contact information from Sticky Note9                                                              |
| YouTube channel for additional tips and tutorials: https://www.youtube.com/@YaronBeen/videos                    | Support and educational content linked in Sticky Note9                                            |
| LinkedIn profile: https://www.linkedin.com/in/yaronbeen/                                                        | Professional profile for workflow creator/support                                                 |
| Bright Data affiliate link: https://get.brightdata.com/1tndi4600b25                                              | Referral link supporting free content creation (Sticky Note5)                                      |
| Workflow concept: Automatically track OpenAI mentions in Medium blogs with sentiment analysis and data storage   | Described in Sticky Note2 and section notes                                                        |
| Beginner tips: Paste URL and execute; AI + MCP handle scraping and analysis with no coding required               | Highlighted in Sticky Notes 1 and 3                                                                 |
| Google Sheets usage: Centralized storage, real-time updates, no manual export needed                             | Emphasized in Sticky Note3                                                                          |

---

**Disclaimer:**  
The provided text is derived exclusively from an automated workflow created with n8n, an integration and automation tool. This process strictly adheres to applicable content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and publicly available.