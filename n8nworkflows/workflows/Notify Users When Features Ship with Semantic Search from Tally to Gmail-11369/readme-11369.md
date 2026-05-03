Notify Users When Features Ship with Semantic Search from Tally to Gmail

https://n8nworkflows.xyz/workflows/notify-users-when-features-ship-with-semantic-search-from-tally-to-gmail-11369


# Notify Users When Features Ship with Semantic Search from Tally to Gmail

### 1. Workflow Overview

This workflow automates the process of notifying users when requested product features ship, leveraging semantic search and AI to draft personalized emails. It targets product teams who collect feature requests (via Tally forms), store embeddings in Supabase, monitor new feature launches (via RSS or manual trigger), match those launches against past requests semantically, and generate Gmail drafts to notify relevant users.

The workflow can be logically divided into three functional blocks:

- **1.1 Capture & Memorize:** Receives feature requests from Tally forms, cleans and maps input data, generates embeddings using OpenAI or Ollama, and stores user data and embeddings in a Supabase vector store.

- **1.2 Trigger & Search:** Starts on a new feature launch either manually or via an RSS feed, extracts and embeds the new feature description, queries Supabase to find semantically similar past feature requests, and identifies users to notify.

- **1.3 Draft Personal Emails:** Loops through matched users, uses an AI agent to draft personalized Gmail emails connecting their requests to the new feature, and creates Gmail drafts without sending them automatically.

---

### 2. Block-by-Block Analysis

#### 1.1 Capture & Memorize

**Overview:**  
This block listens for new feature requests submitted via a Tally form, sanitizes and maps the form data to consistent variables, generates vector embeddings for the request text, and stores all relevant data (user info, original requests, embeddings) in a Supabase vector database for later semantic search.

**Nodes Involved:**  
- Tally Trigger (disabled)  
- Data Cleaner  
- Embeddings OpenAI  
- Default Data Loader  
- Supabase Vector Store  
- Sticky Note4 (documentation)  
- Sticky Note6 (documentation)  
- Sticky Note (for Data Cleaner config)  
- Sticky Note1 (for Tally Trigger config)  
- Sticky Note2 (for embedding and storage setup)

**Node Details:**

- **Tally Trigger**  
  - *Type:* Tally Forms Trigger  
  - *Role:* Entry point for new form submissions  
  - *Config:* Listens to form ID `"gDbZ6P"` (disabled by default)  
  - *Connections:* Outputs to `Data Cleaner`  
  - *Failure Modes:* Disabled; if enabled, possible webhook or credential issues  

- **Data Cleaner**  
  - *Type:* Set node  
  - *Role:* Maps raw form fields to standardized variables: `user_name`, `user_email`, `original_text`  
  - *Config:* Uses expressions to extract specific form responses (e.g., `$json.question_aaaDDD` mapped to `user_name`)  
  - *Connections:* Outputs to `Supabase Vector Store`  
  - *Failure Modes:* Expression evaluation errors if form fields change or missing  

- **Embeddings OpenAI**  
  - *Type:* LangChain OpenAI Embeddings  
  - *Role:* Converts request text into vector embeddings with model `nomic-embed-text:latest`  
  - *Connections:* Outputs embeddings to `Supabase Vector Store` via `ai_embedding` input  
  - *Failure Modes:* API key or model availability issues; rate limits  

- **Default Data Loader**  
  - *Type:* LangChain Document Default Data Loader  
  - *Role:* Prepares document with metadata (`user_name`, `user_email`) for vector storage  
  - *Config:* Loads `original_text` as document content and attaches metadata  
  - *Connections:* Outputs to `Supabase Vector Store` via `ai_document` input  
  - *Failure Modes:* Malformed metadata or missing text fields  

- **Supabase Vector Store**  
  - *Type:* LangChain Supabase Vector Store (version 1.3)  
  - *Role:* Inserts new feature request embeddings and user metadata into `feature_requests` table in Supabase  
  - *Config:* Uses `insert` mode, table name `feature_requests`, Supabase credentials configured  
  - *Connections:* Receives input from `Data Cleaner`, `Embeddings OpenAI`, `Default Data Loader`  
  - *Failure Modes:* Authentication failures, network errors, Supabase schema mismatch  

- **Sticky Notes (Documentation)**  
  - Provide instructions and configuration tips for nodes in this block.

---

#### 1.2 Trigger & Search

**Overview:**  
This block triggers on new feature announcements either manually or via an RSS feed polling weekly, processes the feature description to generate embeddings, and queries Supabase to find up to 10 semantically similar past feature requests with a match threshold of 0.70.

**Nodes Involved:**  
- When clicking ‘Execute workflow’ (Manual Trigger)  
- RSS Feed Trigger  
- Lunch description (Set node for manual feature description)  
- Manage input - Code in JavaScript  
- Generate Embedding (Ollama) (HTTP Request)  
- HTTP Request - Supabase Search  
- Sticky Note5 (dual triggers info)  
- Sticky Note3 (dual triggers info)  
- Sticky Note7 (vector search explanation)  
- Sticky Note (configuration reminders)

**Node Details:**

- **When clicking ‘Execute workflow’**  
  - *Type:* Manual trigger node  
  - *Role:* Allows manual testing or triggering with a preset feature description  
  - *Connections:* Outputs to `Lunch description`  

- **RSS Feed Trigger**  
  - *Type:* RSS Feed Read Trigger  
  - *Role:* Polls a product changelog RSS feed weekly for new entries  
  - *Config:* Feed URL `https://your_blog.url/rss/` (placeholder)  
  - *Connections:* Outputs to `Manage input - Code in JavaScript`  

- **Lunch description**  
  - *Type:* Set node  
  - *Role:* Holds a test feature description string `"we lunched dark mode"` for manual trigger  
  - *Connections:* Outputs to `Manage input - Code in JavaScript`  

- **Manage input - Code in JavaScript**  
  - *Type:* Code node  
  - *Role:* Normalizes input from either RSS or manual trigger, prioritizing content snippet, description, title, or manual text  
  - *Script:* Extracts text into `feature_text` variable for embedding  
  - *Connections:* Outputs to `Generate Embedding (Ollama)`  

- **Generate Embedding (Ollama)**  
  - *Type:* HTTP Request  
  - *Role:* Calls Ollama API (`http://host.docker.internal:11434/api/embeddings`) to generate vector embedding for the feature description  
  - *Config:* POST request with model `nomic-embed-text` and prompt as `feature_text`  
  - *Connections:* Outputs to `HTTP Request - Supabase Search`  

- **HTTP Request - Supabase Search**  
  - *Type:* HTTP Request  
  - *Role:* Calls Supabase RPC `match_feature_requests` to find similar feature requests based on embedding  
  - *Config:*  
    - POST body includes `query_embedding` from Ollama output  
    - Match threshold `0.70`  
    - Max matches `10`  
    - Uses Supabase credentials  
  - *Connections:* Outputs to `Loop Matches`  

- **Sticky Notes (Documentation)**  
  - Explain dual trigger support (RSS/manual), vector search details, and configuration requirements for Supabase URL.

---

#### 1.3 Draft Personal Emails

**Overview:**  
For each matched user from the search, this block uses an AI agent to generate a personalized email connecting the user’s original feature request to the newly launched feature. It then creates a Gmail draft message for review and manual sending.

**Nodes Involved:**  
- Loop Matches (split in batches)  
- AI Agent - Draft text maker (LangChain agent)  
- Structured Output Parser  
- OpenAI Chat Model3 (language model)  
- Create Draft (Gmail node)  
- Done! (NoOp node marking completion)  
- Sticky Note8 (drafting emails info)  
- Sticky Note9 (final status message)

**Node Details:**

- **Loop Matches**  
  - *Type:* SplitInBatches  
  - *Role:* Iterates over matched user records one by one to send through AI drafting  
  - *Connections:* Outputs to both `Done!` (end) and `AI Agent - Draft text maker`  

- **AI Agent - Draft text maker**  
  - *Type:* LangChain Agent  
  - *Role:* Uses AI to draft personalized email text in JSON `{subject, body}` format based on user request, user name, and launched feature description  
  - *Prompt:* Includes context about user name, original request, and launched feature; instructs to write subject and friendly HTML email body  
  - *Config:* Has output parser and fallback enabled, retries on failures with 3-second wait  
  - *Connections:* Outputs to `Create Draft`  
  - *Failure Modes:* API failures, malformed output, rate limits  

- **Structured Output Parser**  
  - *Type:* LangChain Structured Output Parser  
  - *Role:* Parses AI output JSON into structured data fields  
  - *Connections:* Feeds into `AI Agent - Draft text maker` as output parser  

- **OpenAI Chat Model3**  
  - *Type:* LangChain Chat Language Model (OpenAI)  
  - *Role:* Underlying AI model called by agent to generate response  
  - *Config:* Uses model ID `command-r7b:7b-12-2024-q8_0` (likely a custom or specialized OpenAI model)  
  - *Connections:* Connected as language model for `AI Agent - Draft text maker`  

- **Create Draft**  
  - *Type:* Gmail node  
  - *Role:* Creates a draft email in Gmail with AI-generated subject and body, addressed to matched user email  
  - *Config:* Draft resource, subject and message taken dynamically from AI output and matched user data  
  - *Connections:* Outputs to `Loop Matches` (next batch)  

- **Done!**  
  - *Type:* NoOp  
  - *Role:* Marks end of workflow after all matches processed  
  - *Connections:* None  

- **Sticky Notes (Documentation)**  
  - Provide context on drafting emails, final success message, and next steps.

---

### 3. Summary Table

| Node Name                   | Node Type                              | Functional Role                                      | Input Node(s)             | Output Node(s)                  | Sticky Note                                                                                                    |
|-----------------------------|--------------------------------------|-----------------------------------------------------|---------------------------|--------------------------------|----------------------------------------------------------------------------------------------------------------|
| Tally Trigger               | n8n-nodes-tallyforms.tallyTrigger    | Trigger on Tally form submission                     |                           | Data Cleaner                   | "### ⚙️ How to Configure:\n**Tally Trigger:** Connect your form credential."                                  |
| Data Cleaner                | n8n-nodes-base.set                   | Map & clean form data fields                         | Tally Trigger             | Supabase Vector Store          | "### ⚙️ How to Configure:\n**Data Cleaner (Important):** Open this node!\n   - Map your form's \"Name\" field to `user_name`\n   - Map your form's \"Email\" field to `user_email`\n   - Map the \"Request\" field to `original_text`" |
| Embeddings OpenAI           | @n8n/n8n-nodes-langchain.embeddingsOpenAi | Generate vector embedding for feature requests       |                           | Supabase Vector Store (ai_embedding) |                                                                                                                |
| Default Data Loader         | @n8n/n8n-nodes-langchain.documentDefaultDataLoader | Prepare document with metadata for vector storage   |                           | Supabase Vector Store (ai_document) |                                                                                                                |
| Supabase Vector Store       | @n8n/n8n-nodes-langchain.vectorStoreSupabase | Insert embeddings & metadata into Supabase           | Data Cleaner, Embeddings OpenAI, Default Data Loader |                              | "### ⚙️ How to Configure:\nCopy the SQL script from the **Template Description** and run it in Supabase first.\nThis node:\n1. **Embed:** Converts text to vectors (using Nomic/Ollama).\n2. **Store:** Saves user info & vectors to Supabase." |
| Sticky Note4               | n8n-nodes-base.stickyNote             | Documentation block for Capture & Memorize           |                           |                                | "## 📥 1. Capture & Memorize\nThis section cleans form data, maps fields (Name/Email), creates embeddings, and stores them in Supabase." |
| When clicking ‘Execute workflow’ | n8n-nodes-base.manualTrigger          | Manual trigger to test or start feature launch       |                           | Lunch description              | "### Dual Triggers\n- **RSS Feed:** Checks your changelog weekly. If you post your product lunch it triggers.\n- **Manual:** Great for testing! Type a phrase in the 'Lunch description' node to test the search and click \"Execute\" ." |
| RSS Feed Trigger            | n8n-nodes-base.rssFeedReadTrigger    | Weekly polling of RSS feed for new feature launches  |                           | Manage input - Code in JavaScript | See above ("Dual Triggers")                                                                                        |
| Lunch description           | n8n-nodes-base.set                   | Holds manual feature launch description               | When clicking ‘Execute workflow’ | Manage input - Code in JavaScript |                                                                                                                |
| Manage input - Code in JavaScript | n8n-nodes-base.code                   | Normalize description input from RSS or manual       | RSS Feed Trigger, Lunch description | Generate Embedding (Ollama)    |                                                                                                                |
| Generate Embedding (Ollama) | n8n-nodes-base.httpRequest            | Call Ollama API to get vector embedding               | Manage input - Code in JavaScript | HTTP Request - Supabase Search | "### The Vector Search\n- **Embedding:** Converts your announcement into numbers (768 dimensions).\n- **HTTP Request:** Sends those numbers to Supabase to find the nearest semantic match.\n  - *Setup:* Update the **URL** in this node to your Supabase Project URL." |
| HTTP Request - Supabase Search | n8n-nodes-base.httpRequest            | Query Supabase RPC for matching feature requests      | Generate Embedding (Ollama) | Loop Matches                  |                                                                                                                |
| Loop Matches                | n8n-nodes-base.splitInBatches         | Loop over matched users to process individually       | HTTP Request - Supabase Search | Done!, AI Agent - Draft text maker |                                                                                                                |
| AI Agent - Draft text maker | @n8n/n8n-nodes-langchain.agent        | Generate personalized email text via AI               | Loop Matches              | Create Draft                  | "## ✍️ 3. Draft Personal Emails\nIn the loops through every match and AI writes a custom email connecting their specific request to your new feature.\n* **Gmail:** Creates a DRAFT (does not send automatically)." |
| Structured Output Parser    | @n8n/n8n-nodes-langchain.outputParserStructured | Parses AI agent JSON email output                      |                           | AI Agent - Draft text maker (ai_outputParser) |                                                                                                                |
| OpenAI Chat Model3          | @n8n/n8n-nodes-langchain.lmChatOpenAi | AI language model for agent                             |                           | AI Agent - Draft text maker (ai_languageModel) |                                                                                                                |
| Create Draft                | n8n-nodes-base.gmail                  | Create Gmail draft with AI-generated email             | AI Agent - Draft text maker | Loop Matches                  |                                                                                                                |
| Done!                      | n8n-nodes-base.noOp                   | No operation node marking workflow completion          | Loop Matches              |                                | "# 🎉 Done!\n\nIf the workflow reaches here, you have successfully:\n1. Found a user who felt ignored.\n2. Drafted an email showing you listened.\n3. Potentially reduced churn or gained an upsell.\n\n**Next Step:** Check your Gmail Drafts folder! 📮" |
| Sticky Note5               | n8n-nodes-base.stickyNote             | Documentation on dual triggers                          |                           |                                | See Sticky Note3                                                                                               |
| Sticky Note6               | n8n-nodes-base.stickyNote             | Overview & setup instructions                           |                           |                                | "Turn old feature requests into new revenue by notifying users when you ship what they asked for. Instructions and setup tips." |
| Sticky Note7               | n8n-nodes-base.stickyNote             | Explanation of vector search details                    |                           |                                | See Generate Embedding (Ollama) entry                                                                          |
| Sticky Note8               | n8n-nodes-base.stickyNote             | Drafting personal emails block description              |                           |                                | See AI Agent - Draft text maker entry                                                                           |
| Sticky Note9               | n8n-nodes-base.stickyNote             | Final success message block                              |                           |                                | See Done! node entry                                                                                            |
| Sticky Note3               | n8n-nodes-base.stickyNote             | Dual triggers explanation                                |                           |                                | See Sticky Note5                                                                                                |
| Sticky Note                | n8n-nodes-base.stickyNote             | Data Cleaner configuration reminder                      |                           |                                | See Data Cleaner node entry                                                                                      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Tally Trigger node**  
   - Type: `Tally Forms Trigger`  
   - Set form ID to your Tally form (e.g., `"gDbZ6P"`)  
   - Connect output to `Data Cleaner`  
   - *Note:* Disabled by default; enable after configuration  

2. **Create Data Cleaner node**  
   - Type: `Set`  
   - Map form fields to variables:  
     - `user_name` = expression: `{{$json.question_aaaDDD}}` (adjust field IDs)  
     - `user_email` = expression: `{{$json.question_bbbEEE}}`  
     - `original_text` = expression: `{{$json.question_cccFFF}}`  
   - Connect output to `Supabase Vector Store`  

3. **Create Embeddings OpenAI node**  
   - Type: `LangChain Embeddings OpenAI`  
   - Model: `nomic-embed-text:latest`  
   - Connect output as `ai_embedding` input to `Supabase Vector Store`  

4. **Create Default Data Loader node**  
   - Type: `LangChain Document Default Data Loader`  
   - JSON Data: expression `{{$json.original_text}}`  
   - Metadata: assign `user_name` and `user_email` from input JSON  
   - Connect output as `ai_document` input to `Supabase Vector Store`  

5. **Create Supabase Vector Store node**  
   - Type: `LangChain Vector Store Supabase`  
   - Mode: `insert`  
   - Table Name: `feature_requests`  
   - Credentials: configure Supabase API credentials with your project details  
   - Connect inputs from `Data Cleaner` (main), `Embeddings OpenAI` (ai_embedding), and `Default Data Loader` (ai_document)  

6. **Create Manual Trigger node**  
   - Type: `Manual Trigger`  
   - Connect output to `Lunch description`  

7. **Create RSS Feed Trigger node**  
   - Type: `RSS Feed Read Trigger`  
   - Set feed URL to your changelog RSS (e.g., `https://your_blog.url/rss/`)  
   - Polling interval: weekly  
   - Connect output to `Manage input - Code in JavaScript`  

8. **Create Lunch description node**  
   - Type: `Set`  
   - Assign string: `description = "we lunched dark mode"` (or any test feature text)  
   - Connect output to `Manage input - Code in JavaScript`  

9. **Create Manage input - Code in JavaScript node**  
   - Type: `Code`  
   - JS script:  
     ```js
     const input = $input.first().json;
     const text = input.contentSnippet || input.description || input.title || input.manual_text;
     return { feature_text: text };
     ```  
   - Connect output to `Generate Embedding (Ollama)`  

10. **Create Generate Embedding (Ollama) node**  
    - Type: `HTTP Request`  
    - Method: POST  
    - URL: `http://host.docker.internal:11434/api/embeddings` (adjust for your Ollama host)  
    - Body Parameters:  
      - `model`: "nomic-embed-text"  
      - `prompt`: expression `{{$json.feature_text}}`  
    - Connect output to `HTTP Request - Supabase Search`  

11. **Create HTTP Request - Supabase Search node**  
    - Type: `HTTP Request`  
    - Method: POST  
    - URL: your Supabase RPC endpoint, e.g., `https://your_link_to.supabase.co/rest/v1/rpc/match_feature_requests`  
    - Use Supabase API credentials  
    - Body parameters:  
      - `query_embedding`: expression `{{$json.embedding}}`  
      - `match_threshold`: `0.70`  
      - `match_count`: `10`  
    - Connect output to `Loop Matches`  

12. **Create Loop Matches node**  
    - Type: `SplitInBatches`  
    - Default batch size (1)  
    - Connect outputs:  
      - Main output to `Done!` node (end)  
      - Secondary output to `AI Agent - Draft text maker`  

13. **Create AI Agent - Draft text maker node**  
    - Type: `LangChain Agent`  
    - Prompt:  
      ```
      You are a helpful product manager.

      Context:
      - User: {{$json.user_name}}
      - They asked for: "{{$json.content}}"
      - We launched: "{{ $('Lunch description').item.json.description }}"

      Write a JSON object with a subject line and a short, friendly HTML email body telling them the feature is here.

      Format: { "subject": "...", "body": "..." }
      ```  
    - Enable output parser with JSON schema example: `{ "subject": "...", "body": "..." }`  
    - Set retry on fail with 3000 ms wait  
    - Connect output to `Create Draft`  

14. **Create Structured Output Parser node**  
    - Type: `LangChain Output Parser Structured`  
    - JSON schema example: `{ "subject": "...", "body": "..." }`  
    - Connect output as `ai_outputParser` input to `AI Agent - Draft text maker`  

15. **Create OpenAI Chat Model3 node**  
    - Type: `LangChain LM Chat OpenAI`  
    - Model ID: `command-r7b:7b-12-2024-q8_0` or your preferred GPT model  
    - Connect output as `ai_languageModel` input to `AI Agent - Draft text maker`  

16. **Create Create Draft node**  
    - Type: `Gmail`  
    - Action: Create draft (resource: draft)  
    - Subject: expression `{{$json.output.subject}} - to:{{$json.user_email}}`  
    - Message body: expression `{{$json.output.body}}`  
    - Connect output back to `Loop Matches` for next iteration  

17. **Create Done! node**  
    - Type: `NoOp`  
    - Connect input from `Loop Matches` when no more batches  

18. **Add Sticky Notes**  
    - Add descriptive sticky notes for documentation and configuration guidance as per the original workflow for clarity and maintenance.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                 | Context or Link                                                                                         |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------|
| Turn old feature requests into new revenue by notifying users when you ship what they asked for. Setup requires running the provided SQL script in Supabase before storing embeddings.                         | Found in Sticky Note6                                                                                   |
| Dual triggers allow either manual testing with a custom feature description or automated weekly RSS feed polling for new product launches.                                                                     | Sticky Note3, Sticky Note5                                                                              |
| For the vector search, update the HTTP Request node's URL to your Supabase project URL and ensure the Supabase RPC `match_feature_requests` is implemented as per the template description.                     | Sticky Note7, HTTP Request - Supabase Search node                                                     |
| Emails are drafted in Gmail as drafts; they are not sent automatically. Users should review drafts before sending to maintain personalization and control over communication.                                   | Sticky Note8, Create Draft node                                                                        |
| After successful execution, check Gmail Drafts folder for generated emails. This can reduce churn or increase upsell by showing users their feedback was heard and implemented.                                | Sticky Note9, Done! node                                                                               |
| Configure credentials carefully: Supabase API for vector storage and search, OpenAI API for embeddings and chat models, Gmail OAuth2 for draft creation, and Tally credentials for form triggering (if enabled). | Credentials must be properly set in n8n credential manager                                             |
| The Ollama embedding API is expected to run locally or on a reachable host; update the URL if deploying differently.                                                                                           | Generate Embedding (Ollama) node configuration                                                        |

---

**Disclaimer:**  
The text provided is extracted exclusively from an automated workflow created with n8n, an integration and automation tool. This process strictly complies with applicable content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.