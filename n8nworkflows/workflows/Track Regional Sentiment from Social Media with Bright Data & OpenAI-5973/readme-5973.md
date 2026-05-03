Track Regional Sentiment from Social Media with Bright Data & OpenAI

https://n8nworkflows.xyz/workflows/track-regional-sentiment-from-social-media-with-bright-data---openai-5973


# Track Regional Sentiment from Social Media with Bright Data & OpenAI

### 1. Workflow Overview

This workflow automates the process of tracking regional sentiment related to weather by scraping Yelp posts about weather in Los Angeles, analyzing the sentiment of these posts using OpenAI GPT-4, and organizing campaign tasks in Trello based on the insights. It is designed for marketing teams or analysts who want to track weather-related consumer sentiment and tailor campaigns accordingly.

The workflow is organized into three main logical blocks:

- **1.1 Input Reception and Initialization:** Manual trigger and setting the target Yelp URL to scrape weather-related posts in Los Angeles.
- **1.2 AI-Assisted Data Scraping and Sentiment Analysis:** An AI Agent that orchestrates scraping Yelp posts using Bright Data MCP Client, processes the scraped data with OpenAI GPT-4 for sentiment analysis, and structures the data into JSON.
- **1.3 Campaign Task Creation:** Using the structured data to automatically create Trello cards for marketing campaigns tailored to the identified sentiment.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and Initialization

**Overview:**  
This block allows manual activation of the workflow and defines the Yelp URL to be scraped for weather-related posts in Los Angeles.

**Nodes Involved:**  
- 🔘 Trigger: Manual Execution  
- 🌐 Set Yelp URL (Weather Posts - Los Angeles)

**Node Details:**

- **🔘 Trigger: Manual Execution**  
  - *Type:* Manual Trigger  
  - *Role:* Starts the workflow execution manually.  
  - *Configuration:* No parameters set; user initiates manually.  
  - *Connections:* Output to `🌐 Set Yelp URL (Weather Posts - Los Angeles)`.  
  - *Edge Cases:* If not manually triggered, workflow will not run. No inputs required.  
  - *Notes:* Enables flexible start time without automation schedules.

- **🌐 Set Yelp URL (Weather Posts - Los Angeles)**  
  - *Type:* Set Node  
  - *Role:* Defines the URL containing the Yelp search results for weather-related posts in Los Angeles.  
  - *Configuration:* Sets a string variable `URL` with value:  
    `https://www.yelp.com/search?find_desc=weather&find_loc=Los+Angeles%2C+CA%2C+United+States`  
  - *Connections:* Output to `🤖 AI Agent: Scrape Yelp Weather Posts and tailor campaigns`.  
  - *Edge Cases:* URL must be valid to scrape relevant posts; malformed URLs may break scraping.  
  - *Notes:* Easily modifiable to target other cities or topics by changing the URL.

---

#### 1.2 AI-Assisted Data Scraping and Sentiment Analysis

**Overview:**  
This core block uses an AI Agent to scrape Yelp for weather-related posts, perform sentiment analysis, and parse the output into structured JSON for downstream use.

**Nodes Involved:**  
- 🤖 AI Agent: Scrape Yelp Weather Posts and tailor campaigns  
- 🌐 MCP Client: Scrape Weather Posts Data (used inside AI Agent)  
- 💬 AI Model (OpenAI GPT-4) (used inside AI Agent)  
- Auto-fixing Output Parser (used inside AI Agent)  
- 📝 Parse Scraped Data into JSON Format1 (structured output parser)

**Node Details:**

- **🤖 AI Agent: Scrape Yelp Weather Posts and tailor campaigns**  
  - *Type:* LangChain AI Agent Node  
  - *Role:* Coordinates scraping and AI processing. Sends instructions to scrape Yelp posts, extract relevant fields, perform sentiment analysis, and generate marketing campaign suggestions.  
  - *Configuration:*  
    - Prompt instructs the agent to scrape the Yelp URL (provided by input variable `URL`).  
    - Extracts business name, location, rating, review count, post text, weather-related content, sentiment (Positive, Negative, Neutral), and campaign suggestions.  
    - Uses a combination of MCP scraping and OpenAI GPT-4 for processing.  
    - Output is parsed and structured.  
  - *Connections:*  
    - Input from `🌐 Set Yelp URL (Weather Posts - Los Angeles)`  
    - Uses AI language model `💬 AI Model` and MCP Client tool internally.  
    - Outputs to `📋 Create Trello Card for Weather Campaign`.  
  - *Edge Cases:*  
    - Scraping might fail due to network errors, rate limits, or changes in Yelp site structure.  
    - AI model may produce unexpected output if prompt is misunderstood.  
    - Parsing errors if output format changes.  
    - Requires valid API credentials for OpenAI and MCP Client.  
  - *Notes:* This node encapsulates a complex multi-step AI-driven scraping and analysis process.

- **🌐 MCP Client: Scrape Weather Posts Data**  
  - *Type:* Bright Data MCP Client Tool Node  
  - *Role:* Executes the scraping of Yelp pages as markdown to retrieve raw data.  
  - *Configuration:*  
    - Tool name set to `scrape_as_markdown`.  
    - Tool parameters are generated dynamically by AI Agent.  
  - *Connections:* Called internally by AI Agent node as a tool.  
  - *Edge Cases:*  
    - Requires valid Bright Data credentials.  
    - Can fail due to network issues or scraping limits.  
  - *Notes:* Bright Data MCP Client is an external scraping service integrated seamlessly.

- **💬 AI Model (OpenAI GPT-4)**  
  - *Type:* LangChain OpenAI Chat Model Node  
  - *Role:* Processes scraped raw data to extract structured information and perform sentiment analysis.  
  - *Configuration:*  
    - Model: `gpt-4.1-mini` (a GPT-4 variant optimized for this task).  
  - *Connections:* Called internally by AI Agent.  
  - *Edge Cases:*  
    - Requires valid OpenAI API credentials.  
    - Subject to OpenAI rate limits and potential API errors.  
  - *Notes:* Central AI component for understanding and parsing data.

- **Auto-fixing Output Parser**  
  - *Type:* LangChain Output Parser Autofixing Node  
  - *Role:* Automatically corrects and normalizes AI output to conform to expected JSON structure.  
  - *Connections:*  
    - Receives output from `OpenAI Chat Model`.  
    - Sends cleaned output back to `AI Agent`.  
  - *Edge Cases:* May fail if AI output is severely malformed.

- **📝 Parse Scraped Data into JSON Format1**  
  - *Type:* LangChain Structured Output Parser Node  
  - *Role:* Parses the AI agent’s output into a defined JSON schema for use in the workflow.  
  - *Configuration:*  
    - Contains a JSON schema example that defines fields like `business_name`, `location`, `rating`, `reviews_count`, `post_text`, `weather_related`, `sentiment`, `campaign_suggestion`, and nested `trello_card` info.  
  - *Connections:*  
    - Input from AI Agent’s parsed output.  
    - Output to `Auto-fixing Output Parser`.  
  - *Edge Cases:* Schema mismatch or unexpected AI output can cause parsing errors.

---

#### 1.3 Campaign Task Creation

**Overview:**  
This block creates Trello cards for each weather-related campaign based on the structured data, enabling marketing teams to track and manage campaigns efficiently.

**Nodes Involved:**  
- 📋 Create Trello Card for Weather Campaign

**Node Details:**

- **📋 Create Trello Card for Weather Campaign**  
  - *Type:* Trello Node  
  - *Role:* Creates Trello cards with campaign details derived from the AI analysis.  
  - *Configuration:*  
    - Card name constructed dynamically from campaign title in AI output.  
    - Description includes location, rating, review count, sentiment, and campaign details.  
    - No due date set by default.  
  - *Connections:*  
    - Input from `🤖 AI Agent: Scrape Yelp Weather Posts and tailor campaigns`.  
  - *Edge Cases:*  
    - Requires valid Trello OAuth2 credentials.  
    - Failure possible if Trello API rate limits exceeded or auth credentials expire.  
  - *Notes:* Centralizes campaign management tasks for teams.

---

### 3. Summary Table

| Node Name                                       | Node Type                                | Functional Role                              | Input Node(s)                            | Output Node(s)                                  | Sticky Note                                                                                               |
|------------------------------------------------|-----------------------------------------|----------------------------------------------|-----------------------------------------|------------------------------------------------|-----------------------------------------------------------------------------------------------------------|
| 🔘 Trigger: Manual Execution                     | Manual Trigger                         | Workflow manual start trigger                | —                                       | 🌐 Set Yelp URL (Weather Posts - Los Angeles)   | Enables manual start of workflow by user click.                                                           |
| 🌐 Set Yelp URL (Weather Posts - Los Angeles)   | Set Node                              | Sets Yelp search URL variable                 | 🔘 Trigger: Manual Execution             | 🤖 AI Agent: Scrape Yelp Weather Posts and tailor campaigns | Defines the target Yelp URL for scraping weather-related posts in Los Angeles.                            |
| 🤖 AI Agent: Scrape Yelp Weather Posts and tailor campaigns | LangChain AI Agent                     | Orchestrates scraping, sentiment analysis, and campaign suggestion | 🌐 Set Yelp URL (Weather Posts - Los Angeles) | 📋 Create Trello Card for Weather Campaign          | Uses Bright Data MCP Client and OpenAI to scrape and analyze Yelp data into structured JSON.               |
| 🌐 MCP Client: Scrape Weather Posts Data         | Bright Data MCP Client Tool             | Scrapes Yelp page as markdown                 | Called internally by AI Agent            | Called internally by AI Agent                     | Performs the actual data scraping from Yelp using Bright Data service.                                     |
| 💬 AI Model                                      | LangChain OpenAI Chat Model             | Processes scraped data, performs sentiment analysis | Called internally by AI Agent            | Auto-fixing Output Parser                         | Uses GPT-4 model for data extraction, sentiment analysis, and campaign generation.                         |
| Auto-fixing Output Parser                        | LangChain Output Parser Autofixing      | Cleans and normalizes AI output                | 💬 AI Model                             | 🤖 AI Agent: Scrape Yelp Weather Posts and tailor campaigns | Ensures AI output conforms to expected JSON format.                                                       |
| 📝 Parse Scraped Data into JSON Format1          | LangChain Structured Output Parser      | Parses AI output into structured JSON          | 🤖 AI Agent: Scrape Yelp Weather Posts and tailor campaigns | Auto-fixing Output Parser                         | Defines JSON schema for extracted data and campaign information.                                          |
| 📋 Create Trello Card for Weather Campaign       | Trello Node                           | Creates Trello cards for campaign tracking    | 🤖 AI Agent: Scrape Yelp Weather Posts and tailor campaigns | —                                                | Automates Trello card creation for marketing campaigns based on sentiment analysis.                        |
| Sticky Note                                      | Sticky Note                           | Documentation and guidance                     | —                                       | —                                              | Section 1: Input URL & Trigger Workflow explanation.                                                     |
| Sticky Note1                                     | Sticky Note                           | Documentation and guidance                     | —                                       | —                                              | Section 2: Scrape Data and Structure it explanation.                                                     |
| Sticky Note2                                     | Sticky Note                           | Documentation and guidance                     | —                                       | —                                              | Section 3: Create Trello Card for Campaign explanation.                                                  |
| Sticky Note3                                     | Sticky Note                           | Workflow summary and detailed overview         | —                                       | —                                              | Full workflow summary and benefits.                                                                     |
| Sticky Note5                                     | Sticky Note                           | Affiliate link to Bright Data                   | —                                       | —                                              | "I’ll receive a tiny commission if you join Bright Data through this link." [https://get.brightdata.com/1tndi4600b25] |
| Sticky Note9                                     | Sticky Note                           | Workflow support and contact information       | —                                       | —                                              | Contact: Yaron@nofluff.online; Links to YouTube and LinkedIn profiles.                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Add a Manual Trigger node named `🔘 Trigger: Manual Execution`. No parameters needed. This node will start the workflow manually.

2. **Add Set Node for Yelp URL**  
   - Add a Set node named `🌐 Set Yelp URL (Weather Posts - Los Angeles)`.  
   - Configure it to assign one string variable `URL` with value:  
     `https://www.yelp.com/search?find_desc=weather&find_loc=Los+Angeles%2C+CA%2C+United+States`.  
   - Connect output of Manual Trigger to this Set node.

3. **Add LangChain AI Agent Node**  
   - Add a LangChain AI Agent node named `🤖 AI Agent: Scrape Yelp Weather Posts and tailor campaigns`.  
   - In parameters, set the prompt to instruct the agent to:  
     - Scrape the Yelp URL from the variable `URL`.  
     - Extract business name, location, rating, reviews count, post text, weather-related mentions.  
     - Perform sentiment analysis categorizing as Positive, Negative, or Neutral.  
     - Suggest tailored marketing campaigns based on sentiment.  
   - Configure it to accept the URL from the Set node.  
   - Connect output of Set node to this AI Agent node.

4. **Inside the AI Agent node:**  
   - Configure it to use the Bright Data MCP Client Tool node (`scrape_as_markdown`) to scrape Yelp pages.  
   - Add a LangChain OpenAI Chat Model node named `💬 AI Model` with model set to `gpt-4.1-mini`.  
   - Add an Auto-fixing Output Parser node to normalize AI output.  
   - Add a Structured Output Parser node named `📝 Parse Scraped Data into JSON Format1` with the JSON schema example provided.  
   - Connect these internal nodes accordingly to:  
     MCP Client → OpenAI Chat Model → Output Parser → AI Agent output.  
   - Credentials needed:  
     - OpenAI API credentials configured for the Chat Model node.  
     - Bright Data MCP Client credentials configured for the MCP Client node.

5. **Add Trello Node to Create Cards**  
   - Add a Trello node named `📋 Create Trello Card for Weather Campaign`.  
   - Configure with your Trello OAuth2 credentials.  
   - Set card name and description dynamically using expressions based on AI Agent output, for example:  
     - Name: `Campaign {{ $json.output[0].trello_card.title }}`  
     - Description: includes location, rating, reviews, sentiment, campaign suggestion.  
   - Connect output of AI Agent node to this Trello node.

6. **Connect All Nodes**  
   - Connect Manual Trigger → Set Yelp URL → AI Agent → Trello Card.

7. **Set Credentials**  
   - OpenAI API key configured in `💬 AI Model`.  
   - Bright Data MCP Client credentials configured in MCP Client node inside AI Agent.  
   - Trello OAuth2 credentials configured in Trello node.

8. **Test Workflow**  
   - Manually execute the workflow.  
   - Verify Yelp data is scraped, sentiment analyzed, and Trello cards created.

---

### 5. General Notes & Resources

| Note Content                                                                                                                   | Context or Link                                                                                      |
|-------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| I’ll receive a tiny commission if you join Bright Data through this link—thanks for fueling more free content!                 | [https://get.brightdata.com/1tndi4600b25](https://get.brightdata.com/1tndi4600b25)                 |
| For any questions or support, contact: Yaron@nofluff.online. Explore more tips and tutorials on YouTube and LinkedIn:          | YouTube: https://www.youtube.com/@YaronBeen/videos<br>LinkedIn: https://www.linkedin.com/in/yaronbeen/ |
| This workflow automates scraping Yelp weather-related posts, analyzing sentiment, and creating Trello campaign cards.          | Workflow summary and use cases for marketing teams and analysts.                                   |
| Sentiment categories used: Positive, Negative, Neutral. Campaign suggestions tailored accordingly for marketing actions.        | Important for correct campaign targeting based on consumer sentiment.                              |
| The workflow uses Bright Data MCP Client for scraping and OpenAI GPT-4 for AI processing; valid API credentials are mandatory.  | Credentials setup is critical to avoid failures.                                                   |

---

This document provides a comprehensive understanding and reproduction guide for the "Track Regional Sentiment from Social Media with Bright Data & OpenAI" n8n workflow, enabling technical users and AI agents to modify, extend, or deploy it confidently.