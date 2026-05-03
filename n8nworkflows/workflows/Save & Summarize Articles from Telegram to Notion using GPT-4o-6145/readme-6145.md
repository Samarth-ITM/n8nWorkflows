Save & Summarize Articles from Telegram to Notion using GPT-4o

https://n8nworkflows.xyz/workflows/save---summarize-articles-from-telegram-to-notion-using-gpt-4o-6145


# Save & Summarize Articles from Telegram to Notion using GPT-4o

### 1. Workflow Overview

This workflow automates the process of saving and summarizing articles received as links via Telegram messages, using GPT-4o for AI-powered content analysis, and storing the results in a Notion database. It targets users who want to effortlessly collect, summarize, and organize web articles shared in Telegram chats into a structured knowledge base for easy reference.

The workflow is logically divided into the following blocks:

- **1.1 Telegram Input Reception:** Listens for incoming messages in Telegram containing article URLs.
- **1.2 Article Content Retrieval:** Fetches and parses the full article content and title from the URL using a custom-deployed article-parser API.
- **1.3 AI Content Summarization & Tagging:** Uses GPT-4o via Langchain agent to generate a concise highlight and a relevant topic tag.
- **1.4 Metadata Structuring for Notion:** Prepares and organizes data fields extracted and generated to conform with the Notion database schema.
- **1.5 Saving to Notion Database:** Inserts the article data with metadata into the designated Notion knowledge base.
- **1.6 Confirmation Messaging:** Sends a confirmation message back to the Telegram chat with the article summary and Notion page link.

---

### 2. Block-by-Block Analysis

#### 1.1 Telegram Input Reception

- **Overview:**  
  This block listens for new messages in a Telegram bot and triggers the workflow when a message containing an article link is received.

- **Nodes Involved:**  
  - Telegram Trigger

- **Node Details:**

  - **Telegram Trigger**  
    - Type: Telegram Trigger (Webhook-based listener for Telegram updates)  
    - Configuration: Listens specifically for "message" updates.  
    - Credentials: Connected to a Telegram Bot API credential.  
    - Input: Incoming Telegram messages from the configured bot.  
    - Output: Emits the message JSON including chat info and message text.  
    - Potential Failures: Webhook misconfiguration, API rate limits, missing permissions in Telegram Bot.  
    - Notes: Entry point of the workflow.  

#### 1.2 Article Content Retrieval

- **Overview:**  
  This block requests an external article parsing API to extract the article's title and content from the URL sent via Telegram.

- **Nodes Involved:**  
  - Fetch Article Title & Content

- **Node Details:**

  - **Fetch Article Title & Content**  
    - Type: HTTP Request  
    - Configuration: Performs a GET request to the deployed article-parser API URL with the Telegram message text passed as the `url` query parameter.  
    - Key Expression: `https://YOUR_DEPLOYED_ARTICLE_PARSER_URL.vercel.app/api/parse?url={{$json["message"]["text"]}}`  
    - Input: Telegram Trigger output (message text).  
    - Output: JSON with `title` and `content` fields extracted from the article.  
    - Potential Failures: API downtime, invalid or malformed URLs, empty or unparseable content causing downstream AI errors.  
    - Notes: Requires user to deploy the article-parser API externally (e.g., Vercel).  

#### 1.3 AI Content Summarization & Tagging

- **Overview:**  
  Uses GPT-4o via a Langchain agent node to generate a brief highlight summary and a single-word tag describing the article's topic.

- **Nodes Involved:**  
  - Generate Highlight + Tag (AI Agent)

- **Node Details:**

  - **Generate Highlight + Tag (AI Agent)**  
    - Type: Langchain Agent Node (OpenAI chat model wrapper)  
    - Configuration: Sends prompt with article title and content asking for a JSON response containing a highlight and a type tag.  
    - Key Expression (Prompt):  
      ```
      You are a knowledge management assistant. Given an article title and content, your job is to:
      1. Generate a 1–2 sentence highlight summarizing the article.
      2. Suggest a relevant single-word tag for the article's topic (e.g., AI, Health, Startups, Cloud, Finance).
      Here is the article:
      Title: {{ $json["title"] }}
      Content: {{ $json["content"] }}
      Please respond with a JSON object like:
      {
        "highlight": "...",
        "type": "..."
      }
      ```
    - Input: Output from the article parser node (title and content).  
    - Output: AI-generated JSON string with `highlight` and `type`.  
    - Potential Failures: API quota exceeded, malformed output, AI model downtime, parsing issues if response does not follow JSON format strictly.  
    - Notes: The node uses GPT-4o model by default but can be swapped for any other chat model.  

#### 1.4 Metadata Structuring for Notion

- **Overview:**  
  Prepares and structures all necessary metadata fields (title, URL, highlight, tag) into a format suitable for the Notion database insertion.

- **Nodes Involved:**  
  - Structure Metadata for Notion

- **Node Details:**

  - **Structure Metadata for Notion**  
    - Type: Set Node (Data transformation)  
    - Configuration: Extracts and assigns fields:  
      - `title` from article parser output  
      - `url` from Telegram message text  
      - `highlight` and `type` parsed from AI agent output JSON string  
    - Key Expressions:  
      - `={{$node["Fetch Article Title & Content"].json["title"]}}`  
      - `={{$node["Telegram Trigger"].json["message"]["text"]}}`  
      - `={{ JSON.parse($node["Generate Highlight + Tag (AI Agent)"].json["output"].replace(/```json|```/g, '').trim()).highlight }}`  
      - `={{ JSON.parse($node["Generate Highlight + Tag (AI Agent)"].json["output"].replace(/```json|```/g, '').trim()).type }}`  
    - Input: Output of AI agent node and previous nodes.  
    - Output: JSON with normalized properties for Notion insertion.  
    - Potential Failures: JSON parsing errors if AI output is malformed.  

#### 1.5 Saving to Notion Database

- **Overview:**  
  Inserts a new page into the configured Notion database with the article metadata and content.

- **Nodes Involved:**  
  - Save Article to Notion Database

- **Node Details:**

  - **Save Article to Notion Database**  
    - Type: Notion Node (Database Page creation)  
    - Configuration:  
      - Connects to user’s Notion account via OAuth2.  
      - Database selected by URL (user must supply correct Notion database URL).  
      - Page properties set: status (select = Backlog), date added (current date), headline (title), URL, type (multi-select), highlight (rich text).  
    - Key Expressions: Values are derived from the structured metadata node.  
    - Input: Structured metadata.  
    - Output: Notion page creation confirmation including page URL.  
    - Potential Failures: Incorrect Notion credentials, wrong database URL, mismatched property names, API limits.  
    - Notes: User must ensure the Notion database matches the expected schema.  

#### 1.6 Confirmation Messaging

- **Overview:**  
  Sends a confirmation message back to the Telegram chat confirming the article was saved, including the highlight and Notion page link.

- **Nodes Involved:**  
  - Confirm Save via Telegram

- **Node Details:**

  - **Confirm Save via Telegram**  
    - Type: Telegram Node (send message)  
    - Configuration:  
      - Sends a templated message including article title, highlight, and Notion page URL.  
      - Uses chat ID from the original Telegram message to respond in the same chat.  
    - Input: Output from Notion node (page URL) and Telegram trigger (chat ID).  
    - Output: Confirmation message delivered to user.  
    - Potential Failures: Telegram API errors, invalid chat ID, message formatting issues.  

---

### 3. Summary Table

| Node Name                      | Node Type                         | Functional Role                       | Input Node(s)               | Output Node(s)                    | Sticky Note                                                                                                                        |
|-------------------------------|----------------------------------|------------------------------------|-----------------------------|---------------------------------|-----------------------------------------------------------------------------------------------------------------------------------|
| Telegram Trigger               | Telegram Trigger                 | Receive Telegram messages          | —                           | Fetch Article Title & Content    |                                                                                                                                   |
| Fetch Article Title & Content  | HTTP Request                    | Parse article from URL             | Telegram Trigger            | Generate Highlight + Tag (AI Agent) | 👆 Calls your deployed `article-parser-api` to extract a clean **title** + **content** from the article link sent in Telegram.     |
| Generate Highlight + Tag (AI Agent) | Langchain Agent (OpenAI GPT-4o) | Summarize article & suggest tag   | Fetch Article Title & Content | Structure Metadata for Notion     | 👆 You can exchange this with any other chat model of your choice.                                                                |
| Structure Metadata for Notion  | Set                             | Prepare data fields for Notion    | Generate Highlight + Tag (AI Agent) | Save Article to Notion Database   |                                                                                                                                   |
| Save Article to Notion Database| Notion                          | Save article and metadata to Notion | Structure Metadata for Notion | Confirm Save via Telegram        | 👆 How to setup **Save Article to Notion Database** node: connect Notion account, enter database URL, and match database structure. |
| Confirm Save via Telegram      | Telegram                        | Send confirmation message to Telegram | Save Article to Notion Database | —                               |                                                                                                                                   |
| Sticky Note                   | Sticky Note                     | Setup Instructions and Workflow Breakdown | —                           | —                               | 1. Create API credentials (Telegram, OpenAI, Notion). 2. Deploy article parser. 3. Link Notion database. 4. Test and activate.      |
| Sticky Note1                  | Sticky Note                     | Chat model replacement suggestion | —                           | —                               | 👆 You can exchange this with any other chat model of your choice.                                                                |
| Sticky Note2                  | Sticky Note                     | Notion node setup guide            | —                           | —                               | 👆 How to setup **Save Article to Notion Database** node with connection and database URL instructions.                            |
| Sticky Note3                  | Sticky Note                     | Workflow breakdown summary         | —                           | —                               | ### 🔍 Workflow breakdown: from Telegram Trigger to Confirm Save via Telegram.                                                    |
| Sticky Note4                  | Sticky Note                     | Article parser API usage guide     | —                           | —                               | 👆 Calls your deployed `article-parser-api` to extract article content. Configure with your deployment URL and Telegram message. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger Node**  
   - Type: Telegram Trigger  
   - Parameters: Listen for "message" updates.  
   - Credentials: Connect your Telegram Bot API credentials.  
   - Position: Start of the workflow.

2. **Add HTTP Request Node for Article Parsing**  
   - Name: Fetch Article Title & Content  
   - Type: HTTP Request  
   - Parameters:  
     - Method: GET  
     - URL: `https://YOUR_DEPLOYED_ARTICLE_PARSER_URL.vercel.app/api/parse?url={{$json["message"]["text"]}}`  
   - Connect input from Telegram Trigger output.  
   - No credentials needed.  
   - Note: Deploy the article-parser-api externally (e.g., on Vercel).

3. **Add Langchain Agent Node for AI Summarization**  
   - Name: Generate Highlight + Tag (AI Agent)  
   - Type: Langchain Agent Node (OpenAI Chat Model)  
   - Parameters:  
     - Model: Select GPT-4o (or alternate chat models as preferred).  
     - Prompt:  
       ```
       You are a knowledge management assistant. Given an article title and content, your job is to:

       1. Generate a 1–2 sentence highlight summarizing the article.
       2. Suggest a relevant single-word tag for the article's topic (e.g., AI, Health, Startups, Cloud, Finance).

       Here is the article:

       Title: {{ $json["title"] }}

       Content: {{ $json["content"] }}

       Please respond with a JSON object like:

       {
         "highlight": "...",
         "type": "..."
       }
       ```
   - Credentials: Connect your OpenAI API key.  
   - Connect input from HTTP Request node output.

4. **Add Set Node to Structure Metadata**  
   - Name: Structure Metadata for Notion  
   - Type: Set  
   - Parameters: Set fields with expressions:  
     - `title` = `{{$node["Fetch Article Title & Content"].json["title"]}}`  
     - `url` = `{{$node["Telegram Trigger"].json["message"]["text"]}}`  
     - `highlight` = `{{ JSON.parse($node["Generate Highlight + Tag (AI Agent)"].json["output"].replace(/```json|```/g, '').trim()).highlight }}`  
     - `type` = `{{ JSON.parse($node["Generate Highlight + Tag (AI Agent)"].json["output"].replace(/```json|```/g, '').trim()).type }}`  
   - Connect input from AI node output.

5. **Add Notion Node to Save Article**  
   - Name: Save Article to Notion Database  
   - Type: Notion (Create Database Page)  
   - Parameters:  
     - Resource: Database Page  
     - Database ID: Select "By URL" and paste your Notion database URL.  
     - Properties:  
       - Status (select): Backlog  
       - Date Added (date): `{{ $now.format('yyyy-MM-dd') }}`  
       - Headline (title): `{{$json["title"]}}`  
       - URL (url): `{{$json["url"]}}`  
       - Type (multi_select): `{{$json["type"]}}`  
       - Highlight (rich_text): `{{$json["highlight"]}}`  
   - Credentials: Connect your Notion account.  
   - Connect input from Set node output.

6. **Add Telegram Node to Confirm Save**  
   - Name: Confirm Save via Telegram  
   - Type: Telegram (Send Message)  
   - Parameters:  
     - Text:  
       ```
       ✅ New article saved to Notion!

       📌 {{ $json["name"] }}
       📝 Highlight: {{ $json["property_highlight"] }}

       🔗 Notion Page: {{ $json["url"] }}
       ```  
     - Chat ID: `{{$node["Telegram Trigger"].json["message"]["chat"]["id"]}}`  
   - Credentials: Use Telegram Bot API credentials.  
   - Connect input from Notion node output.

7. **Activate the Workflow**  
   - Test by sending an article URL to the Telegram bot.  
   - Verify article parsing, AI summarization, Notion entry creation, and confirmation message.  
   - Adjust credentials, URLs, and database schema as needed.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         | Context or Link                                                                                      |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| Create and connect your API credentials for Telegram Bot, OpenAI API, and Notion Integration.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       | [Telegram Bot Setup](https://sendpulse.com/knowledge-base/chatbot/telegram/create-telegram-chatbot), [OpenAI API Key Guide](https://apidog.com/blog/openai-api-key-free/), [Notion Integration](https://developers.notion.com/docs/create-a-notion-integration#create-your-integration-in-notion) |
| Deploy the article parser API from [article-parser-api GitHub repo](https://github.com/Yosua1011/article-parser-api) to Vercel or any serverless environment.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    | https://github.com/Yosua1011/article-parser-api                                                      |
| Link your Notion database by duplicating the [AI‑Enhanced Knowledge Base Tracker](https://www.notion.com/templates/ai-enhanced-knowledge-base-tracker) template and connect it with n8n as per Notion API documentation.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           | [Notion API Connection Guide](https://www.notion.com/help/add-and-manage-connections-with-the-api?g-exp=use_transcend_for_cookie_consent_on_front--on#add-connections-to-pages) |
| Ensure Notion database properties match expected names and types: "✅ Status" (select), "📅 Date Added" (date), "📰 Headline" (title), "🔗 URL" (url), "🏷️ Type" (multi-select), "✨ Highlight" (rich text).                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             | User must confirm schema compatibility                                                              |
| Test the workflow by clicking Execute Workflow in n8n and sending an article link via Telegram; verify all steps complete successfully before activating.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          |                                                                                                    |
| The AI model used here is GPT-4o, but you can replace it with any other supported OpenAI chat model as needed.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     |                                                                                                    |

---

**Disclaimer:** The provided content exclusively originates from an automated workflow created in n8n, a no-code integration and automation tool. This processing strictly adheres to applicable content policies and contains no illegal, offensive, or restricted material. All processed data is legal and publicly accessible.